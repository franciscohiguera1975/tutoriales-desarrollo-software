# Curso React.js + TypeScript — Página 9B
## Módulo 4 · Integración con APIs
### `fetch`, `axios`, cliente tipado, paginación y CRUD

---

## ¿Qué cubre esta página?

En las páginas anteriores usaste `fetch` dentro de `useEffect` y extrajiste
la lógica a un `useFetch` genérico. Esta página profundiza en el tema:

- Comparativa `fetch` vs `axios`
- Cliente de API tipado y centralizado
- `useFetchData` con soporte para recarga manual (`refetch`)
- `usePagination` — hook de paginación reutilizable
- CRUD completo con `axios` e interceptores
- Variables de entorno con Vite

Lo cubierto aquí aplica cuando **no usas TanStack Query** (página 14).
Si usas TanStack Query, muchos de estos patrones los gestiona la librería.

---

## `fetch` vs `axios`

`fetch` es nativo del navegador — cero dependencias. `axios` es una librería
que simplifica varias tareas habituales.

```ts
// fetch — nativo, sin dependencias
const res  = await fetch('https://api.ejemplo.com/posts')
const data = await res.json()  // hay que parsear manualmente

// ⚠️ fetch NO lanza error en respuestas 4xx/5xx
// Un 404 es una respuesta "exitosa" para fetch — hay que verificar res.ok
if (!res.ok) throw new Error(`HTTP ${res.status}`)
```

```ts
// axios — hay que instalar: npm install axios
import axios from 'axios'

const { data } = await axios.get('https://api.ejemplo.com/posts')
// axios parsea JSON automáticamente
// axios SÍ lanza error en 4xx/5xx — no hace falta verificar
```

### Comparativa directa

| Característica | `fetch` | `axios` |
|---|---|---|
| ¿Nativo? | ✅ Sí — sin instalación | ❌ Requiere `npm install axios` |
| Parseo de JSON | Manual (`res.json()`) | Automático (`res.data`) |
| Errores HTTP 4xx/5xx | ❌ No lanza — verificar `res.ok` | ✅ Lanza automáticamente |
| Interceptores | ❌ No tiene | ✅ `interceptors.request/response` |
| Cancelación | `AbortController` | `AbortController` o `CancelToken` |
| Timeout | Manual con `AbortController` | Opción `timeout` en la config |
| TypeScript | Tipos básicos del DOM | Tipado genérico en `get<T>()` |
| Tamaño | 0 KB | ~14 KB (minificado) |

> **Cuándo usar cada uno**: para proyectos pequeños o cuando el bundle size importa,
> `fetch` es suficiente. Para proyectos con autenticación, interceptores y múltiples
> endpoints, `axios` reduce código repetido significativamente.

---

## Práctica progresiva — de simple a complejo

Los ejemplos avanzan en este orden:

```
1. fetch directo en useEffect                ← lo más simple
2. Cliente fetch centralizado y tipado       ← sin repetición
3. useFetchData con refetch manual           ← hook reutilizable
4. usePagination                             ← hook de UI
5. Cliente axios con interceptores           ← producción real
6. CRUD completo con axios                   ← todo junto
```

---

## 1. `fetch` directo en `useEffect`

El punto de partida — el patrón básico que ya conoces de la página 5,
ahora con tipos explícitos en cada parte:

### `src/components/PostListBasic.tsx`

```tsx
// src/components/PostListBasic.tsx

import { useState, useEffect } from 'react'

interface Post {
  id:     number
  title:  string
  body:   string
  userId: number
}

export default function PostListBasic() {
  const [posts,   setPosts]   = useState<Post[]>([])
  const [loading, setLoading] = useState(true)
  const [error,   setError]   = useState<string | null>(null)

  useEffect(() => {
    let cancelled = false

    async function fetchPosts() {
      try {
        const res = await fetch(
          'https://jsonplaceholder.typicode.com/posts?_limit=5'
        )
        // fetch no lanza en 4xx/5xx — siempre verificar res.ok
        if (!res.ok) throw new Error(`HTTP ${res.status}`)

        const data: Post[] = await res.json()
        if (!cancelled) setPosts(data)
      } catch (err) {
        if (!cancelled) {
          setError(err instanceof Error ? err.message : 'Error desconocido')
        }
      } finally {
        if (!cancelled) setLoading(false)
      }
    }

    fetchPosts()
    return () => { cancelled = true }  // limpieza anti race-condition
  }, [])

  if (loading) return <p style={{ color: '#6b7280' }}>Cargando posts...</p>
  if (error)   return <p style={{ color: '#dc2626' }}>Error: {error}</p>

  return (
    <ul style={{ listStyle: 'none', padding: 0, display: 'flex', flexDirection: 'column', gap: 8 }}>
      {posts.map(post => (
        <li
          key={post.id}
          style={{ padding: '10px 14px', border: '1px solid #e5e7eb', borderRadius: 8 }}
        >
          <p style={{ margin: 0, fontWeight: 600, fontSize: 14 }}>{post.title}</p>
          <p style={{ margin: '4px 0 0', fontSize: 13, color: '#6b7280' }}>
            {post.body.slice(0, 60)}...
          </p>
        </li>
      ))}
    </ul>
  )
}
```

### Prueba esto

- Cambia la URL a `'https://jsonplaceholder.typicode.com/posts?_limit=10'` — observa que la lista crece a 10 ítems sin tocar la lógica de fetching
- Cambia la URL a `'https://jsonplaceholder.typicode.com/posts/9999'` — verifica que `res.ok` es `false`, se lanza el error `"HTTP 404"` y el componente muestra el mensaje de error en rojo
- Elimina la línea `if (!res.ok) throw new Error(...)` y repite la URL con 404 — observa que sin esa verificación, `fetch` no lanza y `data` llega vacía o con forma inesperada
- Añade `console.log('fetching...')` antes del `await fetch(...)` y monta/desmonta el componente rápidamente — confirma que el flag `cancelled` evita actualizaciones de estado tras el desmontaje
- Cambia el tipo `Post[]` a `Post` (singular) en `await res.json()` sin cambiar la URL — TypeScript no lanza error en tiempo de compilación porque `json()` devuelve `any`; observa los errores en runtime al intentar usar `.map`

---

## 2. Cliente `fetch` centralizado y tipado

En lugar de escribir `fetch(...)` en cada componente, se centraliza en
un módulo `src/api/fetchClient.ts`. Una función `request<T>` genérica
maneja el parseo, la verificación de `res.ok` y el tipado de una vez:

### `src/api/fetchClient.ts`

```ts
// src/api/fetchClient.ts

import type { Post, User, Comment } from '../types'

const BASE = 'https://jsonplaceholder.typicode.com'

// Función base genérica — evita repetir fetch + res.ok + res.json()
async function request<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const res = await fetch(`${BASE}${endpoint}`, options)
  if (!res.ok) throw new Error(`HTTP ${res.status} — ${res.statusText}`)
  return res.json() as Promise<T>
}

// Objeto con todos los endpoints tipados
export const fetchApi = {
  // Lectura
  getPosts:    () =>
    request<Post[]>('/posts?_limit=10'),
  getPost:     (id: number) =>
    request<Post>(`/posts/${id}`),
  getUsers:    () =>
    request<User[]>('/users'),
  getUser:     (id: number) =>
    request<User>(`/users/${id}`),
  getComments: (postId: number) =>
    request<Comment[]>(`/posts/${postId}/comments`),

  // Escritura
  createPost: (data: Omit<Post, 'id'>) =>
    request<Post>('/posts', {
      method:  'POST',
      headers: { 'Content-Type': 'application/json' },
      body:    JSON.stringify(data),
    }),
  updatePost: (id: number, data: Partial<Omit<Post, 'id'>>) =>
    request<Post>(`/posts/${id}`, {
      method:  'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body:    JSON.stringify(data),
    }),
  deletePost: (id: number) =>
    request<void>(`/posts/${id}`, { method: 'DELETE' }),
}
```

```tsx
// Uso — limpio, sin fetch manual en el componente
import { fetchApi } from '../api/fetchClient'

const posts = await fetchApi.getPosts()         // Post[]
const user  = await fetchApi.getUser(1)         // User
const post  = await fetchApi.createPost({ ... }) // Post
```

### Prueba esto

- Cambia `BASE` a una URL inventada como `'https://api-falsa.xyz'` — observa que todas las funciones del cliente fallan con el mismo error de red sin tener que cambiar nada más
- Llama `fetchApi.getUser(1)` y `fetchApi.getUser(2)` en el mismo componente — comprueba que ambas llamadas comparten la misma función `request<T>` base y devuelven tipos distintos correctamente tipados
- Añade un nuevo endpoint `getPostComments: (id: number) => request<Comment[]>(`/posts/${id}/comments`)` al objeto `fetchApi` — verifica que TypeScript infiere `Comment[]` sin configuración adicional
- Cambia el endpoint `getPosts` para que use `/posts?_limit=3` — observa que todos los componentes que llamen a `fetchApi.getPosts()` reciben automáticamente solo 3 posts
- Prueba `fetchApi.deletePost(1)` en la consola del navegador — nota que `jsonplaceholder` responde con `{}` y el tipo `void` del cliente no coincide exactamente; observa si TypeScript advierte sobre esto

---

## 3. `useFetchData` — hook con `refetch` manual

La página 9 tenía un `useFetch<T>` básico. Esta versión añade `refetch`:
una función que fuerza una nueva carga sin cambiar la URL.

### `src/hooks/useFetchData.ts`

```ts
// src/hooks/useFetchData.ts

import { useState, useEffect, useCallback } from 'react'

interface FetchState<T> {
  data:    T | null
  loading: boolean
  error:   string | null
  refetch: () => void      // dispara una nueva carga manual
}

export function useFetchData<T>(url: string): FetchState<T> {
  const [data,    setData]    = useState<T | null>(null)
  const [loading, setLoading] = useState(true)
  const [error,   setError]   = useState<string | null>(null)
  const [trigger, setTrigger] = useState(0)

  // Cada vez que se llama, incrementa trigger → re-ejecuta el useEffect
  const refetch = useCallback(() => setTrigger(t => t + 1), [])

  useEffect(() => {
    let cancelled = false
    setLoading(true)
    setError(null)

    async function load() {
      try {
        const res = await fetch(url)
        if (!res.ok) throw new Error(`HTTP ${res.status}`)
        const json: T = await res.json()
        if (!cancelled) { setData(json); setLoading(false) }
      } catch (err) {
        if (!cancelled) {
          setError(err instanceof Error ? err.message : 'Error desconocido')
          setLoading(false)
        }
      }
    }

    load()
    return () => { cancelled = true }
  }, [url, trigger])   // trigger en dependencias — refetch re-ejecuta el efecto

  return { data, loading, error, refetch }
}
```

### `src/components/UserListWithRefetch.tsx`

```tsx
// src/components/UserListWithRefetch.tsx

import { useFetchData } from '../hooks/useFetchData'

interface User {
  id:    number
  name:  string
  email: string
}

export default function UserListWithRefetch() {
  const { data: users, loading, error, refetch } = useFetchData<User[]>(
    'https://jsonplaceholder.typicode.com/users'
  )

  return (
    <div style={{ maxWidth: 440 }}>
      <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 12 }}>
        <span style={{ fontSize: 14, color: '#6b7280' }}>
          {loading ? 'Cargando...' : `${users?.length ?? 0} usuarios`}
        </span>
        <button
          onClick={refetch}
          disabled={loading}
          style={{
            padding: '6px 14px', border: '1px solid #d1d5db', borderRadius: 6,
            background: '#fff', cursor: loading ? 'not-allowed' : 'pointer', fontSize: 13,
          }}
        >
          {loading ? 'Cargando...' : '↺ Recargar'}
        </button>
      </div>

      {error && <p style={{ color: '#dc2626', fontSize: 13 }}>Error: {error}</p>}

      {users && (
        <ul style={{ listStyle: 'none', padding: 0, display: 'flex', flexDirection: 'column', gap: 6 }}>
          {users.map(user => (
            <li
              key={user.id}
              style={{
                display: 'flex', justifyContent: 'space-between',
                padding: '8px 12px', border: '1px solid #e5e7eb', borderRadius: 8,
              }}
            >
              <span style={{ fontWeight: 500, fontSize: 14 }}>{user.name}</span>
              <span style={{ fontSize: 13, color: '#6b7280' }}>{user.email}</span>
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

### Prueba esto

- Haz clic en "↺ Recargar" — observa que el botón se desactiva durante la carga y la lista se refresca con los mismos datos (o datos distintos si el servidor cambió)
- Haz clic en "↺ Recargar" varias veces seguidas rápidamente — verifica que el flag `cancelled` descarta las respuestas anteriores y solo aplica la última gracias al incremento de `trigger`
- Cambia la URL a `'https://jsonplaceholder.typicode.com/posts'` (100 posts) — observa que `useFetchData<Post[]>` maneja cualquier tipo genérico sin cambios en el hook
- Modifica el hook para que `refetch` también reciba una nueva URL como parámetro opcional — comprueba que al pasarla, el `useEffect` vuelve a ejecutarse con la nueva URL
- Añade un timestamp del último fetch junto al botón usando `Date.now()` — observa que se actualiza cada vez que se llama a `refetch`
- Simula un error pasando una URL inválida y haz clic en "↺ Recargar" — verifica que el estado `error` se actualiza y `loading` vuelve a `false` correctamente

---

## 4. `usePagination` — hook de paginación reutilizable

Separa la lógica de paginación del fetching. Funciona con cualquier array,
independientemente de cómo se obtuvieron los datos.

### `src/hooks/usePagination.ts`

```ts
// src/hooks/usePagination.ts

import { useState } from 'react'

interface UsePaginationOptions {
  totalItems: number
  pageSize:   number
}

interface UsePaginationReturn {
  currentPage: number
  totalPages:  number
  goToPage:    (page: number) => void
  nextPage:    () => void
  prevPage:    () => void
  startIndex:  number
  endIndex:    number
  canGoNext:   boolean
  canGoPrev:   boolean
}

export function usePagination({
  totalItems,
  pageSize,
}: UsePaginationOptions): UsePaginationReturn {
  const [currentPage, setCurrentPage] = useState(1)

  const totalPages = Math.max(1, Math.ceil(totalItems / pageSize))
  const startIndex = (currentPage - 1) * pageSize
  const endIndex   = Math.min(startIndex + pageSize, totalItems)

  function goToPage(page: number) {
    setCurrentPage(Math.min(Math.max(1, page), totalPages))
  }

  return {
    currentPage,
    totalPages,
    goToPage,
    nextPage:  () => goToPage(currentPage + 1),
    prevPage:  () => goToPage(currentPage - 1),
    startIndex,
    endIndex,
    canGoNext: currentPage < totalPages,
    canGoPrev: currentPage > 1,
  }
}
```

### `src/components/PaginatedUserList.tsx`

```tsx
// src/components/PaginatedUserList.tsx

import { useFetchData }  from '../hooks/useFetchData'
import { usePagination } from '../hooks/usePagination'

interface User { id: number; name: string; email: string }

const PAGE_SIZE = 3

export default function PaginatedUserList() {
  const { data: users, loading, error } = useFetchData<User[]>(
    'https://jsonplaceholder.typicode.com/users'
  )

  const {
    currentPage, totalPages,
    nextPage, prevPage, goToPage,
    startIndex, endIndex,
    canGoNext, canGoPrev,
  } = usePagination({ totalItems: users?.length ?? 0, pageSize: PAGE_SIZE })

  const pageItems = users?.slice(startIndex, endIndex) ?? []

  if (loading) return <p style={{ color: '#6b7280' }}>Cargando...</p>
  if (error)   return <p style={{ color: '#dc2626' }}>Error: {error}</p>

  return (
    <div style={{ maxWidth: 440 }}>
      <p style={{ fontSize: 13, color: '#6b7280', marginBottom: 8 }}>
        Mostrando {startIndex + 1}–{endIndex} de {users?.length ?? 0} usuarios
      </p>

      <ul style={{ listStyle: 'none', padding: 0, display: 'flex', flexDirection: 'column', gap: 6, marginBottom: 14 }}>
        {pageItems.map(user => (
          <li
            key={user.id}
            style={{ padding: '10px 14px', border: '1px solid #e5e7eb', borderRadius: 8 }}
          >
            <p style={{ margin: 0, fontWeight: 600, fontSize: 14 }}>{user.name}</p>
            <p style={{ margin: 0, fontSize: 12, color: '#6b7280' }}>{user.email}</p>
          </li>
        ))}
      </ul>

      {/* Controles de paginación con números */}
      <div style={{ display: 'flex', alignItems: 'center', gap: 6 }}>
        <button onClick={prevPage} disabled={!canGoPrev} style={pgBtn(!canGoPrev)}>
          ← Ant.
        </button>

        {Array.from({ length: totalPages }, (_, i) => i + 1).map(n => (
          <button
            key={n}
            onClick={() => goToPage(n)}
            style={{
              ...pgBtn(false),
              background:  n === currentPage ? '#0070f3' : '#fff',
              color:       n === currentPage ? '#fff'    : '#374151',
              borderColor: n === currentPage ? '#0070f3' : '#d1d5db',
              fontWeight:  n === currentPage ? 600 : 400,
            }}
          >
            {n}
          </button>
        ))}

        <button onClick={nextPage} disabled={!canGoNext} style={pgBtn(!canGoNext)}>
          Sig. →
        </button>
      </div>
    </div>
  )
}

function pgBtn(disabled: boolean): React.CSSProperties {
  return {
    padding: '6px 12px', borderRadius: 6,
    border: '1px solid #d1d5db',
    background: '#fff',
    color:   disabled ? '#9ca3af' : '#374151',
    cursor:  disabled ? 'not-allowed' : 'pointer',
    fontSize: 13,
  }
}
```

### Prueba esto

- Cambia `PAGE_SIZE` de `3` a `1` — observa que ahora cada página muestra un solo usuario y el número total de páginas aumenta a 10
- Cambia `PAGE_SIZE` a `10` — verifica que con 10 usuarios todos caben en una sola página y los botones de paginación quedan desactivados
- Haz clic en un número de página directamente (por ejemplo "3") — confirma que `goToPage(3)` salta directamente sin pasar por las páginas intermedias
- Haz clic en "← Ant." cuando estás en la primera página — observa que el botón está desactivado y `canGoPrev` es `false`
- Pasa `totalItems: 0` al hook — verifica que `totalPages` devuelve `1` (no `0`) gracias a `Math.max(1, ...)` y no aparecen páginas negativas
- Conecta `usePagination` con un array local de strings en lugar de datos del servidor — confirma que el hook es agnóstico al origen de los datos y funciona igual

---

## 5. Cliente `axios` con interceptores

### Instalación

```bash
npm install axios
```

### `src/api/axiosClient.ts`

```ts
// src/api/axiosClient.ts

import axios, { type AxiosInstance, type AxiosResponse } from 'axios'
import type { Post, User } from '../types'

// Instancia centralizada — configuración compartida por todos los endpoints
const api: AxiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_URL ?? 'https://jsonplaceholder.typicode.com',
  timeout: 8000,
  headers: { 'Content-Type': 'application/json' },
})

// Interceptor de request — añade el token de auth en cada llamada
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Interceptor de response — manejo centralizado de errores HTTP
api.interceptors.response.use(
  (response: AxiosResponse) => response,
  (error: unknown) => {
    if (axios.isAxiosError(error)) {
      const status = error.response?.status

      if (status === 401) {
        // Token expirado — redirigir al login
        localStorage.removeItem('token')
        window.location.href = '/login'
      }
      if (status === 404) console.warn('Recurso no encontrado')
      if (status === 500) console.warn('Error interno del servidor')

      // Lanza un error con el mensaje del servidor si existe
      throw new Error(
        (error.response?.data as { message?: string })?.message
        ?? error.message
      )
    }
    throw error
  }
)

// Endpoints tipados — el genérico en get<T>/post<T> tipará r.data automáticamente
export const axiosApi = {
  getPosts:   () =>
    api.get<Post[]>('/posts?_limit=10').then(r => r.data),
  getPost:    (id: number) =>
    api.get<Post>(`/posts/${id}`).then(r => r.data),
  getUsers:   () =>
    api.get<User[]>('/users').then(r => r.data),
  createPost: (data: Omit<Post, 'id'>) =>
    api.post<Post>('/posts', data).then(r => r.data),
  updatePost: (id: number, data: Partial<Omit<Post, 'id'>>) =>
    api.patch<Post>(`/posts/${id}`, data).then(r => r.data),
  deletePost: (id: number) =>
    api.delete<void>(`/posts/${id}`).then(r => r.data),
}

export default api
```

### Prueba esto

- Guarda `'fake-token-123'` en `localStorage.setItem('token', 'fake-token-123')` desde la consola del navegador, recarga y realiza cualquier llamada — observa en la pestaña Network de DevTools que la cabecera `Authorization: Bearer fake-token-123` aparece en la petición
- Cambia `timeout: 8000` a `timeout: 1` (1 ms) — observa que casi todas las peticiones fallan con un error de timeout de axios
- Modifica el interceptor de respuesta para que haga `console.log(error.response?.status)` antes de lanzar — verifica que el código de estado HTTP aparece en la consola para cualquier error 4xx/5xx
- Cambia `baseURL` para apuntar a un servidor que devuelva un `401` — observa que el interceptor llama a `localStorage.removeItem('token')` y redirige a `/login`
- Añade `console.log('request:', config.url)` en el interceptor de request — observa en la consola que cada llamada a `axiosApi.*` registra su URL antes de ser enviada

---

## 6. CRUD completo con `axios`

### `src/components/PostCrudWithAxios.tsx`

```tsx
// src/components/PostCrudWithAxios.tsx

import { useState, useEffect } from 'react'
import { axiosApi }             from '../api/axiosClient'

interface Post {
  id:     number
  title:  string
  body:   string
  userId: number
}

export default function PostCrudWithAxios() {
  const [posts,   setPosts]   = useState<Post[]>([])
  const [loading, setLoading] = useState(true)
  const [error,   setError]   = useState<string | null>(null)
  const [title,   setTitle]   = useState('')
  const [saving,  setSaving]  = useState(false)

  // Carga inicial
  useEffect(() => {
    axiosApi.getPosts()
      .then(data => { setPosts(data); setLoading(false) })
      .catch(err  => { setError(err instanceof Error ? err.message : 'Error'); setLoading(false) })
  }, [])

  // Crear
  async function handleCreate(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    if (!title.trim()) return
    setSaving(true)
    try {
      const newPost = await axiosApi.createPost({ title: title.trim(), body: '', userId: 1 })
      setPosts(prev => [newPost, ...prev])   // añade al principio de la lista
      setTitle('')
      setError(null)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Error al crear')
    } finally {
      setSaving(false)
    }
  }

  // Eliminar
  async function handleDelete(id: number) {
    try {
      await axiosApi.deletePost(id)
      setPosts(prev => prev.filter(p => p.id !== id))
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Error al eliminar')
    }
  }

  if (loading) return <p style={{ color: '#6b7280' }}>Cargando...</p>

  return (
    <div style={{ maxWidth: 440 }}>
      {error && (
        <p style={{ margin: '0 0 12px', fontSize: 13, color: '#dc2626' }}>{error}</p>
      )}

      {/* Formulario de creación */}
      <form onSubmit={handleCreate} style={{ display: 'flex', gap: 8, marginBottom: 16 }}>
        <input
          value={title}
          onChange={e => setTitle(e.target.value)}
          placeholder="Título del nuevo post..."
          disabled={saving}
          style={{
            flex: 1, padding: '8px 12px',
            border: '1px solid #d1d5db', borderRadius: 6, fontSize: 14,
          }}
        />
        <button
          type="submit"
          disabled={saving || !title.trim()}
          style={{
            padding: '8px 14px', background: '#0070f3', color: '#fff',
            border: 'none', borderRadius: 6, cursor: 'pointer', fontWeight: 500,
          }}
        >
          {saving ? '...' : 'Crear'}
        </button>
      </form>

      {/* Lista con eliminación */}
      <div style={{ display: 'flex', flexDirection: 'column', gap: 8 }}>
        {posts.map(post => (
          <div
            key={post.id}
            style={{
              display: 'flex', justifyContent: 'space-between', alignItems: 'center',
              padding: '10px 14px', border: '1px solid #e5e7eb', borderRadius: 8,
            }}
          >
            <p style={{ margin: 0, fontSize: 14, flex: 1 }}>{post.title}</p>
            <button
              onClick={() => handleDelete(post.id)}
              style={{
                marginLeft: 10, padding: '4px 8px',
                background: '#fef2f2', color: '#dc2626',
                border: 'none', borderRadius: 4, cursor: 'pointer', fontSize: 12,
              }}
            >
              ✕
            </button>
          </div>
        ))}
      </div>
    </div>
  )
}
```

### Prueba esto

- Escribe un título y haz clic en "Crear" — observa que el nuevo post aparece al principio de la lista de forma optimista (antes de que el servidor confirme) y el input se vacía
- Deja el input vacío y haz clic en "Crear" — verifica que la función retorna sin hacer la petición gracias a `if (!title.trim()) return`
- Haz clic en "✕" junto a cualquier post — observa que desaparece de la lista local instantáneamente (la API de jsonplaceholder simula la eliminación pero no persiste)
- Crea un post y recarga la página — comprueba que el post creado desaparece, ya que `jsonplaceholder` no guarda los datos entre sesiones
- Intenta crear un post mientras `saving` es `true` (haz doble clic muy rápido en "Crear") — verifica que el botón queda desactivado durante la petición y no se envían duplicados
- Modifica `handleDelete` para que muestre un `window.confirm` antes de eliminar — observa que el flujo asíncrono funciona igual dentro del `try/catch`

---

## Variables de entorno con Vite

En proyectos reales la URL de la API no va hardcodeada en el código.
Vite expone variables de entorno con el prefijo `VITE_`:

```bash
# .env.development
VITE_API_URL=https://api.desarrollo.com

# .env.production
VITE_API_URL=https://api.produccion.com
```

```ts
// Acceso desde el código
const BASE = import.meta.env.VITE_API_URL

// Con valor por defecto si la variable no existe
const BASE = import.meta.env.VITE_API_URL ?? 'https://jsonplaceholder.typicode.com'
```

> El prefijo `VITE_` es obligatorio. Variables sin ese prefijo no son
> accesibles desde el código del navegador — solo desde la configuración
> de Vite en Node.js.

Para que TypeScript conozca las variables de entorno personalizadas,
añádelas en `src/vite-env.d.ts`:

```ts
// src/vite-env.d.ts

/// <reference types="vite/client" />

interface ImportMetaEnv {
  readonly VITE_API_URL: string
  // añade aquí todas tus variables VITE_*
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

---

## `src/App.tsx`

```tsx
// src/App.tsx

import PostListBasic       from './components/PostListBasic'
import UserListWithRefetch from './components/UserListWithRefetch'
import PaginatedUserList   from './components/PaginatedUserList'
import PostCrudWithAxios   from './components/PostCrudWithAxios'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  PostListBasic       — fetch directo en useEffect                │
// │  2  UserListWithRefetch — useFetchData con refetch manual           │
// │  3  PaginatedUserList   — paginación con usePagination              │
// │  4  PostCrudWithAxios   — CRUD completo con axios                   │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content =
    PASO === 1 ? <PostListBasic /> :
    PASO === 2 ? <UserListWithRefetch /> :
    PASO === 3 ? <PaginatedUserList /> :
    PASO === 4 ? <PostCrudWithAxios /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 600, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

---

## Cuándo usar qué

| Situación | Herramienta recomendada |
|---|---|
| Aprendiendo, proyecto pequeño | `fetch` directo en `useEffect` |
| Varios componentes que fetchen la misma URL | `useFetchData` con caché manual |
| App con auth, tokens e interceptores | `axios` con cliente centralizado |
| App con muchos endpoints y caché inteligente | TanStack Query (página 14) |

---

## Ejercicios propuestos

1. **Tabla de usuarios con búsqueda** — usa `useFetchData` para cargar usuarios
   y filtra el array en memoria con un input de texto. Sin llamadas al servidor
   por cada letra.

2. **Botón de recarga con timestamp** — modifica `UserListWithRefetch` para que
   muestre la hora de la última carga junto al botón de refetch.

3. **Paginación con URL** — combina `usePagination` con `useSearchParams` de
   React Router para que la página actual se guarde en `?page=2`. Al recargar
   el navegador, la paginación debe restaurarse.

4. **Manejo de timeout** — modifica `useFetchData` para aceptar un parámetro
   `timeoutMs` y usar `AbortController` + `setTimeout` para cancelar la petición
   si tarda demasiado.

5. **Cliente axios con reintentos** — añade un interceptor de respuesta que reintente
   automáticamente las peticiones fallidas hasta 3 veces con un delay de 1 segundo
   entre cada intento.

---

## Resumen de la página 9B

- `fetch` no lanza errores en respuestas 4xx/5xx — siempre verificar `res.ok`.
- `axios` lanza automáticamente, parsea JSON y permite interceptores globales.
- Un cliente centralizado (`fetchClient.ts` o `axiosClient.ts`) evita repetir URL base, headers y manejo de errores en cada componente.
- `useFetchData` con `trigger` + `useCallback` implementa `refetch` sin cambiar la URL.
- `usePagination` separa la lógica de navegación del fetching — funciona con cualquier array.
- Los interceptores de `axios` son el lugar correcto para manejar tokens de autenticación y errores HTTP globales.
- Las variables de entorno en Vite van en `.env.*` con prefijo `VITE_` y se acceden con `import.meta.env.VITE_*`.
- Para proyectos con múltiples endpoints y necesidad de caché, TanStack Query (página 14) reemplaza `useFetchData` con ventajas significativas.

---

> Esta página se ubica entre la página 9 (hooks personalizados)
> y la página 10 (React Router), como complemento antes de la arquitectura.