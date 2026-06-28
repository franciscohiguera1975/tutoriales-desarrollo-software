# ShopApp React Web — Módulo 2

## Infraestructura — Axios client, interceptors JWT y LocalStorage

**Duración estimada:** 2–3 horas
**Prerequisitos:** Módulo 1 completado, backend `shopapi_02` desplegado y corriendo en `http://localhost:8000`.

---

> **Objetivo**
> Construir la capa de infraestructura de datos de ShopApp: un wrapper tipado sobre localStorage
> para persistir tokens JWT, una clase de error personalizada para manejar respuestas de la API,
> y un cliente Axios con interceptors que añaden el token Bearer en cada petición y que
> gestionan automáticamente el refresco de sesión cuando el access token expira.
>
> **Checkpoint final**
> - `localStore.getAccessToken()` devuelve el token guardado en `shopapp_access`.
> - `apiClient.get('/products/')` funciona sin configuración adicional.
> - Cuando el access token expira, el interceptor de respuesta refresca el token y reintenta la petición original de forma transparente.
> - Si el refresh también falla, se dispara el evento `authExpired` en `window` y la promesa rechaza con un `ApiError`.

---

## 2.1 LocalStorage wrapper

**Archivo:** `src/data/storage/local.storage.ts`

El wrapper encapsula el acceso directo a `localStorage` detrás de una API tipada. Centralizar las
claves y la serialización evita que distintas partes de la app lean o escriban tokens con nombres
inconsistentes y hace sencillo reemplazar el mecanismo de persistencia en el futuro (por ejemplo,
migrar a `sessionStorage` o a cookies HttpOnly).

```typescript
// src/data/storage/local.storage.ts

/** Forma de los tokens guardados en localStorage. */
export interface LocalTokens {
  access: string
  refresh: string
}

/** Claves usadas para guardar los tokens. */
const KEYS = {
  ACCESS: 'shopapp_access',
  REFRESH: 'shopapp_refresh',
} as const

/**
 * Wrapper tipado sobre localStorage para manejar tokens JWT.
 * Todos los métodos son sincrónicos — localStorage es síncrono por spec.
 */
export const localStore = {
  /** Devuelve ambos tokens si existen, null si falta alguno. */
  getTokens(): LocalTokens | null {
    const access = localStorage.getItem(KEYS.ACCESS)
    const refresh = localStorage.getItem(KEYS.REFRESH)
    if (!access || !refresh) return null
    return { access, refresh }
  },

  /** Persiste el access token y el refresh token. */
  setTokens(access: string, refresh: string): void {
    localStorage.setItem(KEYS.ACCESS, access)
    localStorage.setItem(KEYS.REFRESH, refresh)
  },

  /** Elimina ambos tokens (logout o expiración de sesión). */
  clearTokens(): void {
    localStorage.removeItem(KEYS.ACCESS)
    localStorage.removeItem(KEYS.REFRESH)
  },

  /** Devuelve solo el access token, o null si no existe. */
  getAccessToken(): string | null {
    return localStorage.getItem(KEYS.ACCESS)
  },

  /** Devuelve solo el refresh token, o null si no existe. */
  getRefreshToken(): string | null {
    return localStorage.getItem(KEYS.REFRESH)
  },
}
```

El objeto `localStore` usa un literal de objeto en lugar de una clase porque no necesita instancias
múltiples — localStorage es un singleton global. Las claves están centralizadas en `KEYS` para
que un cambio de nombre no requiera buscar cadenas dispersas por todo el proyecto.

---

## 2.2 API error types

**Archivo:** `src/data/api/api.error.ts`

Django REST Framework devuelve errores en varios formatos: un string en `detail` para errores
genéricos y un objeto `{ campo: [lista de mensajes] }` para errores de validación de campos.
La clase `ApiError` normaliza ambos casos en un único tipo que cualquier parte de la app puede
inspeccionar sin parsear manualmente la respuesta HTTP.

```typescript
// src/data/api/api.error.ts
import type { AxiosError } from 'axios'

/** Forma esperada de la respuesta de error de la API Django REST. */
interface DjangoErrorResponse {
  detail?: string
  non_field_errors?: string[]
  [field: string]: string[] | string | undefined
}

/**
 * Error tipado que representa una respuesta de error de la API.
 * Extiende Error nativo para mantener compatibilidad con instanceof
 * y para que el stack trace sea correcto.
 */
export class ApiError extends Error {
  /** Código de estado HTTP (p. ej. 400, 401, 403, 404, 500). */
  status: number
  /** Mensaje principal legible por el usuario. */
  detail: string
  /** Errores por campo del formulario (solo en 400 de validación). */
  fieldErrors?: Record<string, string[]>

  constructor(
    status: number,
    detail: string,
    fieldErrors?: Record<string, string[]>,
  ) {
    super(detail)
    this.name = 'ApiError'
    this.status = status
    this.detail = detail
    this.fieldErrors = fieldErrors
  }
}

/**
 * Convierte cualquier error (Axios o desconocido) en un ApiError normalizado.
 * Úsalo en los bloques catch de los interceptors y de las funciones de la API.
 *
 * @example
 * try {
 *   await apiClient.post('/auth/login/', body)
 * } catch (err) {
 *   throw parseApiError(err)
 * }
 */
export function parseApiError(error: unknown): ApiError {
  // Error de red (sin respuesta del servidor)
  const axiosErr = error as AxiosError<DjangoErrorResponse>
  if (!axiosErr.response) {
    return new ApiError(0, 'No se pudo conectar con el servidor. Verifica tu conexión.')
  }

  const { status, data } = axiosErr.response

  // Extraer el mensaje principal
  let detail = `Error ${status}`
  if (data?.detail) {
    detail = String(data.detail)
  } else if (data?.non_field_errors?.length) {
    detail = data.non_field_errors[0]
  }

  // Extraer errores por campo (validación 400)
  let fieldErrors: Record<string, string[]> | undefined
  if (status === 400 && data) {
    fieldErrors = {}
    for (const [key, value] of Object.entries(data)) {
      if (key === 'detail' || key === 'non_field_errors') continue
      if (Array.isArray(value)) {
        fieldErrors[key] = value.map(String)
      }
    }
    if (Object.keys(fieldErrors).length === 0) {
      fieldErrors = undefined
    }
  }

  return new ApiError(status, detail, fieldErrors)
}
```

`parseApiError` recibe `unknown` —el tipo correcto de la variable en un bloque `catch`— y devuelve
siempre un `ApiError`, garantizando que el llamador nunca tenga que manejar tipos heterogéneos.
Los errores de validación de campo se extraen en `fieldErrors` para que los formularios puedan
mostrarlos junto a los inputs correspondientes con React Hook Form.

---

## 2.3 Axios client

**Archivo:** `src/data/api/api.client.ts`

El cliente Axios centraliza toda la configuración HTTP: la URL base del backend, el timeout, el
header de autorización JWT y la lógica de refresco automático de tokens. El interceptor de request
añade el `Authorization: Bearer <access>` header antes de cada petición. El interceptor de response
detecta los 401 (token expirado) y, de forma transparente, pide un nuevo access token usando el
refresh token y reintenta la petición original sin que el llamador lo sepa.

```typescript
// src/data/api/api.client.ts
import axios, { type AxiosRequestConfig } from 'axios'
import { API_CONFIG } from '@/core/config/api.config'
import { localStore } from '@/data/storage/local.storage'
import { ApiError, parseApiError } from '@/data/api/api.error'

/**
 * Evento personalizado que se dispara en window cuando la sesión expira
 * de forma irrecuperable (refresh token inválido o caducado).
 * El AuthStore escucha este evento para limpiar el estado de usuario.
 */
export const AUTH_EXPIRED_EVENT = 'authExpired'

/** Tipo del detalle del evento authExpired. */
export interface AuthExpiredEventDetail {
  reason: string
}

// ─── Instancia Axios ─────────────────────────────────────────────────────────

/**
 * Cliente HTTP principal de la aplicación.
 * Todas las llamadas a la API deben usar esta instancia, no axios directamente.
 */
export const apiClient = axios.create({
  baseURL: API_CONFIG.BASE_URL,
  timeout: API_CONFIG.TIMEOUT,
  headers: {
    'Content-Type': 'application/json',
  },
})

// ─── Request interceptor ─────────────────────────────────────────────────────

/**
 * Añade el header Authorization: Bearer <access> a todas las peticiones
 * si hay un access token disponible en localStorage.
 * Las rutas públicas ignoran el header si no hay token.
 */
apiClient.interceptors.request.use(
  (config) => {
    const token = localStore.getAccessToken()
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config
  },
  (error) => Promise.reject(parseApiError(error)),
)

// ─── Response interceptor ────────────────────────────────────────────────────

/**
 * Flag para evitar múltiples intentos de refresh simultáneos.
 * Si varias peticiones fallan con 401 al mismo tiempo, solo la primera
 * dispara el refresh; las demás esperan a que se complete.
 */
let isRefreshing = false

/** Cola de callbacks que esperan el nuevo access token. */
let refreshSubscribers: Array<(token: string) => void> = []

function subscribeTokenRefresh(cb: (token: string) => void) {
  refreshSubscribers.push(cb)
}

function notifySubscribers(token: string) {
  refreshSubscribers.forEach((cb) => cb(token))
  refreshSubscribers = []
}

/**
 * Dispara el evento authExpired en window con la razón del fallo.
 * El AuthStore reacciona limpiando el estado de usuario.
 */
function dispatchAuthExpired(reason: string) {
  const event = new CustomEvent<AuthExpiredEventDetail>(AUTH_EXPIRED_EVENT, {
    detail: { reason },
  })
  window.dispatchEvent(event)
}

apiClient.interceptors.response.use(
  // Respuesta exitosa: pasar sin modificar
  (response) => response,

  // Error: manejar 401 con refresh automático
  async (error) => {
    const originalRequest = error.config as AxiosRequestConfig & { _retry?: boolean }

    // Solo intentar refresh en 401 y si no es ya un reintento
    if (error.response?.status !== 401 || originalRequest._retry) {
      return Promise.reject(parseApiError(error))
    }

    // Marcar como reintento para evitar bucle infinito
    originalRequest._retry = true

    const refreshToken = localStore.getRefreshToken()
    if (!refreshToken) {
      // No hay refresh token: sesión perdida definitivamente
      localStore.clearTokens()
      dispatchAuthExpired('No refresh token available')
      return Promise.reject(
        new ApiError(401, 'Sesión expirada. Por favor inicia sesión de nuevo.'),
      )
    }

    if (isRefreshing) {
      // Ya hay un refresh en curso: encolar esta petición
      return new Promise<string>((resolve) => {
        subscribeTokenRefresh(resolve)
      }).then((newToken) => {
        if (originalRequest.headers) {
          originalRequest.headers.Authorization = `Bearer ${newToken}`
        }
        return apiClient(originalRequest)
      })
    }

    // Iniciar el refresh
    isRefreshing = true

    try {
      const { data } = await axios.post<{ access: string }>(
        `${API_CONFIG.BASE_URL}/auth/token/refresh/`,
        { refresh: refreshToken },
        { timeout: API_CONFIG.TIMEOUT },
      )

      const newAccessToken = data.access

      // Guardar el nuevo access token (el refresh token no cambia en simple-jwt por defecto)
      localStore.setTokens(newAccessToken, refreshToken)

      // Notificar a las peticiones en cola
      notifySubscribers(newAccessToken)

      // Reintentar la petición original con el nuevo token
      if (originalRequest.headers) {
        originalRequest.headers.Authorization = `Bearer ${newAccessToken}`
      }

      return apiClient(originalRequest)
    } catch (refreshError) {
      // El refresh token también falló: sesión irrecuperable
      localStore.clearTokens()
      refreshSubscribers = []
      dispatchAuthExpired('Refresh token invalid or expired')

      return Promise.reject(
        new ApiError(401, 'Tu sesión ha expirado. Por favor inicia sesión de nuevo.'),
      )
    } finally {
      isRefreshing = false
    }
  },
)
```

El cliente exporta una única instancia `apiClient` que toda la app importa. El mecanismo de cola
(`refreshSubscribers`) resuelve la condición de carrera en la que varias peticiones concurrentes
reciben un 401 simultáneamente: solo la primera dispara el refresh y las demás esperan a que
termine antes de reintentarse, evitando así múltiples llamadas paralelas a `/auth/token/refresh/`.

### Diagrama del flujo del interceptor de respuesta

```
Petición → Servidor
        ↓
    ¿status 401?
     No → retornar respuesta ✓
     Sí ↓
    ¿_retry = true? → rechazar (evita bucle)
     No ↓
    ¿hay refreshToken?
     No → clearTokens + dispatchAuthExpired + reject
     Sí ↓
    ¿isRefreshing = true?
     Sí → encolar en refreshSubscribers y esperar
     No ↓
    POST /auth/token/refresh/
     Éxito → setTokens + notifySubscribers + reintentar original ✓
     Fallo → clearTokens + dispatchAuthExpired + reject
```

---

## 2.4 Verificar el setup

Antes de implementar el módulo de autenticación, puedes verificar que el cliente Axios funciona
correctamente usando el endpoint público de productos.

### Prueba desde la consola del navegador

Arranca la app con `npm run dev`, abre `http://localhost:5173` y ejecuta lo siguiente en la
consola del navegador (DevTools → Console):

```javascript
// Importar directamente no es posible desde la consola del navegador,
// pero puedes verificar con fetch para confirmar que el backend está listo:
fetch('http://localhost:8000/api/products/')
  .then(r => r.json())
  .then(data => console.log('Productos:', data))
  .catch(err => console.error('Error:', err))
```

### Prueba desde un componente temporal

Añade temporalmente este efecto en `src/App.tsx` para probar el cliente durante el desarrollo:

```typescript
// src/App.tsx — SOLO PARA VERIFICACIÓN, eliminar después
import { useEffect } from 'react'
import AppRouter from './presentation/router/AppRouter'
import { apiClient } from './data/api/api.client'

export default function App() {
  useEffect(() => {
    apiClient.get('/products/').then((res) => {
      console.log('[apiClient] Productos:', res.data)
    }).catch((err) => {
      console.error('[apiClient] Error:', err)
    })
  }, [])

  return <AppRouter />
}
```

Deberías ver en la consola la lista de productos del backend. Si ves un error de CORS, verifica que
`CORS_ALLOWED_ORIGINS` en `settings.py` del backend incluya `http://localhost:5173`.

> Recuerda eliminar el `useEffect` de prueba antes de continuar con el siguiente módulo.

### Verificar que TypeScript no tiene errores

```bash
npx tsc --noEmit
```

No debe haber errores. Si aparece uno relacionado con `axios`, verifica que instalaste la versión
correcta: `npm install axios`.

---

## 2.5 Estructura de archivos del módulo

```
src/
├── core/
│   └── config/
│       └── api.config.ts          ← (módulo 1) BASE_URL y TIMEOUT
├── data/
│   ├── api/
│   │   ├── api.error.ts           ← ApiError + parseApiError
│   │   └── api.client.ts          ← instancia axios + interceptors JWT
│   └── storage/
│       └── local.storage.ts       ← wrapper localStorage tipado
```

---

## Resumen

| Archivo | Responsabilidad |
|---|---|
| `src/data/storage/local.storage.ts` | Persiste y recupera los tokens JWT en localStorage con claves tipadas |
| `src/data/api/api.error.ts` | Normaliza cualquier error HTTP en un `ApiError` con `status`, `detail` y `fieldErrors` |
| `src/data/api/api.client.ts` | Instancia Axios con interceptor de request (añade Bearer token) e interceptor de respuesta (refresh automático y evento `authExpired`) |
