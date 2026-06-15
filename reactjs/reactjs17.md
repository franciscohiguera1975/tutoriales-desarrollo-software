# Curso React.js + TypeScript — Página 14
## Módulo 6 · Ecosistema
### TanStack Query v5: fetching, caché, mutaciones y paginación

---

## ¿Por qué TanStack Query?

Cuando haces fetch de datos en React con `useEffect` + `useState`, terminas
repitiendo siempre el mismo código: estado de carga, estado de error, caché
manual, revalidación, cancelación. TanStack Query (antes React Query)
resuelve todo esto de forma automática y declarativa.

| Sin TanStack Query | Con TanStack Query |
|---|---|
| `loading`, `error`, `data` manuales | Desestructuras `isPending`, `isError`, `data` |
| Caché manual — mismo dato se refetcha en cada componente | Caché automático por `queryKey` |
| Re-fetching manual al volver al foco | Revalidación automática al volver a la pestaña |
| Race conditions con flag `cancelled` | Manejadas internamente |
| Estado de mutaciones gestionado a mano | `useMutation` con `isPending`, `isError`, `isSuccess` |

---

## Instalación

```bash
npm install @tanstack/react-query
```

La versión actual es **v5** — tiene diferencias de API respecto a v4.
El curso usa v5 directamente.

---

## Configuración en `main.tsx`

```tsx
// src/main.tsx

import { StrictMode }          from 'react'
import { createRoot }          from 'react-dom/client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import './index.css'
import App from './App.tsx'

// QueryClient centraliza el caché y la configuración global
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime:  5 * 60 * 1000, // 5 minutos — datos frescos sin re-fetch
      retry:      2,              // reintentos en caso de error
      refetchOnWindowFocus: true, // revalidar al volver a la pestaña
    },
  },
})

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </StrictMode>,
)
```

`QueryClientProvider` debe envolver toda la app — igual que `BrowserRouter`.
Si usas ambos, anida uno dentro del otro en `main.tsx`.

### Prueba esto

- Cambia `staleTime: 5 * 60 * 1000` a `staleTime: 0` en `defaultOptions` — todas las queries se consideran obsoletas inmediatamente y se re-validan en cada focus de ventana
- Cambia `retry: 2` a `retry: 0` — cuando una query falla, el error aparece de inmediato sin reintentos
- Cambia `refetchOnWindowFocus: true` a `refetchOnWindowFocus: false` — minimiza y vuelve a la ventana; observa que TanStack Query ya no hace fetch al recuperar el foco
- Añade `gcTime: 1000` a `defaultOptions.queries` — los datos se eliminan del caché 1 segundo después de que el componente se desmonta
- Abre la pestaña Network de DevTools y cambia de tab del navegador varias veces — observa las peticiones de re-validación automática controladas por `refetchOnWindowFocus`

---

## Capa de API — funciones de fetch tipadas

Las funciones de fetching van en archivos separados en `src/api/`.
Son funciones TypeScript puras — sin hooks, sin estado de React.
TanStack Query las llama cuando necesita los datos.

### `src/api/posts.ts`

```ts
// src/api/posts.ts

export interface Post {
  id:     number
  title:  string
  body:   string
  userId: number
}

export interface NewPost {
  title:  string
  body:   string
  userId: number
}

const BASE = 'https://jsonplaceholder.typicode.com'

export async function fetchPosts(): Promise<Post[]> {
  const res = await fetch(`${BASE}/posts?_limit=10`)
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
  return res.json()
}

export async function fetchPost(id: number): Promise<Post> {
  const res = await fetch(`${BASE}/posts/${id}`)
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
  return res.json()
}

export async function createPost(data: NewPost): Promise<Post> {
  const res = await fetch(`${BASE}/posts`, {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify(data),
  })
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
  return res.json()
}

export async function updatePost(id: number, data: Partial<NewPost>): Promise<Post> {
  const res = await fetch(`${BASE}/posts/${id}`, {
    method:  'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify(data),
  })
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
  return res.json()
}

export async function deletePost(id: number): Promise<void> {
  const res = await fetch(`${BASE}/posts/${id}`, { method: 'DELETE' })
  if (!res.ok) throw new Error(`HTTP ${res.status}`)
}
```

### Prueba esto

- Cambia la constante `BASE` a una URL inválida (`https://api.invalid`) — observa cómo los componentes que usen estas funciones muestran el estado `isError` con el mensaje de red
- Cambia `_limit=10` a `_limit=3` en `fetchPosts` — la lista muestra solo 3 posts; el caché almacena exactamente esa respuesta
- Añade `console.log('fetch ejecutado')` dentro de `fetchPosts` — nota cuántas veces aparece en consola comparado con cuántas veces se renderiza el componente
- Modifica `fetchPost` para que lance siempre un error: `throw new Error('Fallo forzado')` — observa el comportamiento de retry en la consola de red antes de mostrar el error
- Cambia el método de `createPost` de `'POST'` a `'GET'` — la mutation falla y `mutation.isError` se activa mostrando el mensaje de error en el formulario

---

## `useQuery` — leer datos

`useQuery` es el hook principal. Recibe una `queryKey` que identifica
el dato en el caché y una `queryFn` que lo obtiene.

```tsx
const { data, isPending, isError, error, isSuccess, isFetching } = useQuery({
  queryKey: ['posts'],
  queryFn:  fetchPosts,
})
```

### Estados de `useQuery` en v5

| Estado | Significado |
|---|---|
| `isPending` | Primera carga — no hay datos en caché todavía |
| `isFetching` | Hace un fetch en background (incluye re-validaciones) |
| `isError` | El último fetch falló |
| `isSuccess` | Hay datos disponibles |
| `isStale` | Los datos están disponibles pero se consideran obsoletos |

> `isPending` reemplaza a `isLoading` de v4. En v5 `isLoading` sigue
> existiendo pero `isPending` es el nombre canónico.

---

### `src/components/PostList.tsx`

```tsx
// src/components/PostList.tsx

import { useQuery } from '@tanstack/react-query'
import { fetchPosts, type Post } from '../api/posts'

export default function PostList() {
  const {
    data:      posts,
    isPending,
    isError,
    error,
    isFetching,
  } = useQuery({
    queryKey:  ['posts'],
    queryFn:   fetchPosts,
    staleTime: 5 * 60 * 1000,  // 5 minutos sin re-fetch
  })

  if (isPending) {
    return (
      <div style={{ display: 'flex', flexDirection: 'column', gap: 8 }}>
        {[...Array(5)].map((_, i) => (
          <div
            key={i}
            style={{
              height: 60, background: '#f3f4f6',
              borderRadius: 8, animation: 'pulse 1.5s infinite',
            }}
          />
        ))}
      </div>
    )
  }

  if (isError) {
    return (
      <div style={{ padding: 16, background: '#fef2f2', borderRadius: 8, color: '#dc2626' }}>
        Error al cargar posts: {error.message}
      </div>
    )
  }

  return (
    <div>
      {isFetching && (
        <p style={{ fontSize: 12, color: '#9ca3af', marginBottom: 8 }}>
          Actualizando...
        </p>
      )}
      <div style={{ display: 'flex', flexDirection: 'column', gap: 10 }}>
        {posts.map((post: Post) => (
          <div
            key={post.id}
            style={{ padding: '12px 16px', border: '1px solid #e5e7eb', borderRadius: 8 }}
          >
            <p style={{ margin: 0, fontWeight: 600, fontSize: 14 }}>{post.title}</p>
            <p style={{ margin: '4px 0 0', fontSize: 13, color: '#6b7280' }}>
              {post.body.slice(0, 80)}...
            </p>
          </div>
        ))}
      </div>
    </div>
  )
}
```

### Prueba esto

- Cambia `staleTime: 5 * 60 * 1000` a `staleTime: 0` — cambia de tab y vuelve; observa el indicador "Actualizando..." aparecer mientras re-valida en background
- Cambia `staleTime: 5 * 60 * 1000` a `staleTime: Infinity` — los datos nunca se consideran obsoletos y no se re-validan aunque cambies de tab repetidamente
- Modifica el array de skeletons de `[...Array(5)]` a `[...Array(10)]` — el estado de carga muestra 10 placeholders en lugar de 5
- Remueve la condición `isFetching` y su párrafo — cambia de tab y vuelve; los datos se actualizan silenciosamente sin ninguna indicación visual
- Abre DevTools Network, recarga la página y observa la única petición a `jsonplaceholder`; cambia de tab y vuelve — aparece una segunda petición de re-validación después de los 5 minutos de `staleTime`
- Añade `refetchInterval: 3000` a las opciones de `useQuery` — observa "Actualizando..." aparecer cada 3 segundos sin interacción del usuario

---

### `src/components/PostDetail.tsx`

`queryKey` con parámetros — cada combinación de valores es una entrada
separada en el caché. `enabled` evita el fetch cuando el ID no es válido.

```tsx
// src/components/PostDetail.tsx

import { useQuery } from '@tanstack/react-query'
import { fetchPost } from '../api/posts'

interface PostDetailProps {
  postId: number
}

export default function PostDetail({ postId }: PostDetailProps) {
  const { data: post, isPending, isError, error } = useQuery({
    queryKey: ['post', postId],      // clave única por postId
    queryFn:  () => fetchPost(postId),
    enabled:  postId > 0,            // no hace fetch si postId es 0 o negativo
    staleTime: 10 * 60 * 1000,
  })

  if (!postId) return <p style={{ color: '#9ca3af' }}>Selecciona un post para ver el detalle.</p>
  if (isPending) return <p>Cargando post #{postId}...</p>
  if (isError)   return <p style={{ color: '#dc2626' }}>Error: {error.message}</p>

  return (
    <article style={{ padding: 20, border: '1px solid #e5e7eb', borderRadius: 10 }}>
      <p style={{ margin: '0 0 4px', fontSize: 12, color: '#9ca3af' }}>Post #{post.id}</p>
      <h2 style={{ margin: '0 0 12px', fontSize: 18 }}>{post.title}</h2>
      <p style={{ margin: 0, color: '#374151', lineHeight: 1.6 }}>{post.body}</p>
    </article>
  )
}
```

### Prueba esto

- Selecciona el post 1, luego el 2, luego vuelve al 1 — observa en Network que el detalle del post 1 no se re-fetcha; se sirve desde el caché con `queryKey: ['post', 1]`
- Cambia `enabled: postId > 0` a `enabled: false` — el componente nunca hace fetch aunque `postId` tenga un valor válido; siempre muestra "Cargando post #N..."
- Cambia `staleTime: 10 * 60 * 1000` a `staleTime: 0` — navega entre posts y vuelve al primero; observa que cada visita dispara una nueva petición de red
- Añade `retry: 5` a las opciones — modifica `fetchPost` para lanzar un error y observa en consola los 5 reintentos antes de mostrar el mensaje de error
- Cambia `queryKey: ['post', postId]` a `queryKey: ['post']` — todos los posts comparten la misma entrada del caché; seleccionar post 2 sobreescribe los datos del post 1

---

## `useMutation` — crear, actualizar, eliminar

`useMutation` gestiona operaciones que modifican datos en el servidor.

### `src/components/CreatePostForm.tsx`

```tsx
// src/components/CreatePostForm.tsx

import { useState }                         from 'react'
import { useMutation, useQueryClient }      from '@tanstack/react-query'
import { createPost, type NewPost }         from '../api/posts'

export default function CreatePostForm() {
  const queryClient = useQueryClient()
  const [title, setTitle] = useState('')
  const [body,  setBody]  = useState('')

  const mutation = useMutation({
    mutationFn: (data: NewPost) => createPost(data),

    onSuccess: () => {
      // Invalida el caché de 'posts' — fuerza un re-fetch
      queryClient.invalidateQueries({ queryKey: ['posts'] })
      setTitle('')
      setBody('')
    },

    onError: (error) => {
      console.error('Error al crear post:', error.message)
    },
  })

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    if (!title.trim() || !body.trim()) return
    mutation.mutate({ title, body, userId: 1 })
  }

  return (
    <form
      onSubmit={handleSubmit}
      style={{ display: 'flex', flexDirection: 'column', gap: 10, maxWidth: 400 }}
    >
      <input
        value={title}
        onChange={(e) => setTitle(e.target.value)}
        placeholder="Título del post"
        disabled={mutation.isPending}
        style={{ padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6 }}
      />
      <textarea
        value={body}
        onChange={(e) => setBody(e.target.value)}
        placeholder="Contenido del post"
        rows={3}
        disabled={mutation.isPending}
        style={{ padding: '8px 12px', border: '1px solid #d1d5db', borderRadius: 6, resize: 'vertical', fontFamily: 'inherit' }}
      />

      {mutation.isError && (
        <p style={{ margin: 0, fontSize: 13, color: '#dc2626' }}>
          Error: {mutation.error.message}
        </p>
      )}
      {mutation.isSuccess && (
        <p style={{ margin: 0, fontSize: 13, color: '#16a34a' }}>
          ✅ Post creado correctamente
        </p>
      )}

      <button
        type="submit"
        disabled={mutation.isPending || !title.trim()}
        style={{
          padding: '10px',
          background: mutation.isPending ? '#93c5fd' : '#0070f3',
          color: '#fff', border: 'none', borderRadius: 6,
          cursor: mutation.isPending ? 'not-allowed' : 'pointer',
          fontWeight: 500,
        }}
      >
        {mutation.isPending ? 'Creando...' : 'Crear post'}
      </button>
    </form>
  )
}
```

### Prueba esto

- Completa el formulario y envía — observa el botón cambiar a "Creando..." con fondo azul claro mientras `mutation.isPending` es true
- Deja el campo título vacío y envía — el botón permanece deshabilitado y la mutation nunca se llama gracias a la validación `!title.trim()`
- Envía el formulario y observa en Network la petición POST — aunque `jsonplaceholder` no guarda los datos realmente, responde con el post creado y `invalidateQueries` dispara un re-fetch de la lista
- Elimina `queryClient.invalidateQueries(...)` del `onSuccess` — crea un post y observa que la lista no se actualiza porque el caché de `['posts']` no se invalida
- Cambia `userId: 1` a `userId: 999` en `mutation.mutate(...)` — el post se crea igualmente; `jsonplaceholder` acepta cualquier userId
- Añade `onSettled: () => console.log('Mutation terminada')` — observa en consola que se ejecuta tanto en éxito como en error

---

### `src/components/PostListWithDelete.tsx`

Eliminación con **actualización optimista** — el item desaparece
de la UI inmediatamente, antes de que el servidor confirme.
Si el servidor falla, se revierte el estado anterior.

```tsx
// src/components/PostListWithDelete.tsx

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { fetchPosts, deletePost, type Post }      from '../api/posts'

export default function PostListWithDelete() {
  const queryClient = useQueryClient()

  const { data: posts, isPending } = useQuery({
    queryKey: ['posts'],
    queryFn:  fetchPosts,
  })

  const deleteMutation = useMutation({
    mutationFn: deletePost,

    // Actualización optimista — se ejecuta ANTES de la llamada al servidor
    onMutate: async (deletedId: number) => {
      // Cancela cualquier re-fetch en curso para no sobreescribir el optimismo
      await queryClient.cancelQueries({ queryKey: ['posts'] })

      // Guarda el estado anterior para rollback si falla
      const previousPosts = queryClient.getQueryData<Post[]>(['posts'])

      // Actualiza el caché inmediatamente (optimista)
      queryClient.setQueryData<Post[]>(['posts'], (old) =>
        old?.filter((post) => post.id !== deletedId) ?? []
      )

      // Retorna el contexto con el estado previo
      return { previousPosts }
    },

    // Rollback si el servidor devuelve error
    onError: (_err, _deletedId, context) => {
      if (context?.previousPosts) {
        queryClient.setQueryData(['posts'], context.previousPosts)
      }
    },

    // Revalida siempre al terminar (éxito o error)
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['posts'] })
    },
  })

  if (isPending) return <p>Cargando...</p>

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 8 }}>
      {posts?.map((post) => (
        <div
          key={post.id}
          style={{
            display: 'flex', justifyContent: 'space-between', alignItems: 'center',
            padding: '10px 16px', border: '1px solid #e5e7eb', borderRadius: 8,
          }}
        >
          <span style={{ fontSize: 14 }}>{post.title}</span>
          <button
            onClick={() => deleteMutation.mutate(post.id)}
            disabled={deleteMutation.isPending}
            style={{
              padding: '4px 10px', background: '#fef2f2', color: '#dc2626',
              border: '1px solid #fecaca', borderRadius: 6,
              cursor: 'pointer', fontSize: 12,
            }}
          >
            Eliminar
          </button>
        </div>
      ))}
    </div>
  )
}
```

### Prueba esto

- Haz clic en "Eliminar" en cualquier post — el item desaparece de la lista instantáneamente antes de que el servidor responda; eso es la actualización optimista en `onMutate`
- Elimina el bloque `onError` completo — si la petición fallara, el item desaparecería permanentemente de la UI sin restaurarse; el rollback es esencial para la UX
- Elimina `await queryClient.cancelQueries(...)` del `onMutate` — si hay un re-fetch en curso cuando eliminas, puede sobreescribir el estado optimista y el item reaparecería brevemente
- Añade `console.log('onMutate', deletedId)` en `onMutate` y `console.log('onSettled')` en `onSettled` — observa el orden de ejecución: onMutate → (petición) → onSettled
- Cambia `onSettled` para que llame a `invalidateQueries` con `queryKey: ['posts', 'paginated', 1]` en lugar de `['posts']` — la lista principal no se re-valida; el caché de paginación tampoco

---

## Paginación con `keepPreviousData`

```tsx
// src/components/PaginatedPosts.tsx

import { useState }                         from 'react'
import { useQuery, keepPreviousData }       from '@tanstack/react-query'
import { type Post }                        from '../api/posts'

const PAGE_SIZE = 5

export default function PaginatedPosts() {
  const [page, setPage] = useState(1)

  const { data: posts, isPending, isPlaceholderData } = useQuery({
    queryKey: ['posts', 'paginated', page],
    queryFn:  async () => {
      const res = await fetch(
        `https://jsonplaceholder.typicode.com/posts?_page=${page}&_limit=${PAGE_SIZE}`
      )
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      return res.json() as Promise<Post[]>
    },
    // keepPreviousData — muestra datos de la página anterior mientras carga la siguiente
    // evita el parpadeo de "loading..." en cada cambio de página
    placeholderData: keepPreviousData,
    staleTime: 60 * 1000,
  })

  return (
    <div style={{ maxWidth: 480 }}>
      <div
        style={{
          display: 'flex', flexDirection: 'column', gap: 8,
          opacity: isPlaceholderData ? 0.6 : 1,
          transition: 'opacity 0.2s',
        }}
      >
        {isPending
          ? <p style={{ color: '#9ca3af' }}>Cargando...</p>
          : posts?.map((post) => (
              <div
                key={post.id}
                style={{ padding: '10px 14px', border: '1px solid #e5e7eb', borderRadius: 8 }}
              >
                <p style={{ margin: 0, fontSize: 13, fontWeight: 600 }}>{post.title}</p>
              </div>
            ))
        }
      </div>

      <div style={{ display: 'flex', alignItems: 'center', gap: 12, marginTop: 16 }}>
        <button
          onClick={() => setPage((p) => Math.max(1, p - 1))}
          disabled={page === 1}
          style={pageBtn(page === 1)}
        >
          ← Anterior
        </button>

        <span style={{ fontSize: 14, color: '#374151' }}>
          Página <strong>{page}</strong>
        </span>

        <button
          onClick={() => setPage((p) => p + 1)}
          disabled={isPlaceholderData}
          style={pageBtn(!!isPlaceholderData)}
        >
          Siguiente →
        </button>
      </div>
    </div>
  )
}

function pageBtn(disabled: boolean): React.CSSProperties {
  return {
    padding: '6px 14px', borderRadius: 6,
    border: '1px solid #d1d5db',
    background: disabled ? '#f9fafb' : '#fff',
    color: disabled ? '#9ca3af' : '#374151',
    cursor: disabled ? 'not-allowed' : 'pointer',
  }
}
```

### Prueba esto

- Navega a la página 2 y observa la transición — sin parpadeo de "Cargando..." porque `placeholderData: keepPreviousData` mantiene los datos anteriores visibles con `opacity: 0.6`
- Elimina `placeholderData: keepPreviousData` — navega entre páginas y observa el parpadeo de contenido vacío mientras llegan los nuevos datos
- Cambia `PAGE_SIZE` de `5` a `2` — cada página muestra solo 2 posts; navega para ver que cada combinación `['posts', 'paginated', page]` es una entrada separada en el caché
- Cambia `opacity: isPlaceholderData ? 0.6 : 1` a `opacity: isPlaceholderData ? 0.2 : 1` — la transición entre páginas es visualmente más obvia
- Añade `staleTime: 0` — navega adelante y atrás entre páginas y observa en Network que cada visita a una página ya cacheada dispara una petición de re-validación en background
- Desactiva el botón "Siguiente" con `disabled={!posts || posts.length < PAGE_SIZE}` en lugar de `disabled={isPlaceholderData}` — el botón se deshabilita cuando la última página tiene menos posts que `PAGE_SIZE`

---

## `src/App.tsx`

```tsx
// src/App.tsx

import PostList           from './components/PostList'
import PostDetail         from './components/PostDetail'
import CreatePostForm     from './components/CreatePostForm'
import PostListWithDelete from './components/PostListWithDelete'
import PaginatedPosts     from './components/PaginatedPosts'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  PostList           — useQuery, lista con caché y skeleton       │
// │  2  PostDetail         — useQuery con parámetro y enabled           │
// │  3  CreatePostForm     — useMutation con invalidateQueries          │
// │  4  PostListWithDelete — useMutation con actualización optimista     │
// │  5  PaginatedPosts     — paginación con keepPreviousData            │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content =
    PASO === 1 ? <PostList /> :
    PASO === 2 ? <PostDetail postId={1} /> :
    PASO === 3 ? <CreatePostForm /> :
    PASO === 4 ? <PostListWithDelete /> :
    PASO === 5 ? <PaginatedPosts /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 600, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

---

## `queryKey` — la clave del caché

La `queryKey` es un array que identifica de forma única cada dato en el caché.
Su diseño determina cuándo React Query re-usa datos vs cuándo hace un nuevo fetch.

```ts
// Dato global sin parámetros
queryKey: ['posts']

// Dato con un parámetro — una entrada por userId
queryKey: ['posts', { userId }]

// Dato con múltiples parámetros
queryKey: ['posts', 'paginated', page]

// Detalle de un recurso específico
queryKey: ['post', postId]

// Dato con filtros
queryKey: ['products', { category, sort, page }]
```

Cuando `invalidateQueries({ queryKey: ['posts'] })` se llama, invalida
**todas** las queries cuya clave empieza con `['posts']` — incluyendo
`['posts', 'paginated', 1]`, `['posts', { userId: 5 }]`, etc.

---

## Diferencias clave v4 → v5

| v4 | v5 |
|---|---|
| `isLoading` | `isPending` (canónico) — `isLoading` sigue funcionando |
| `keepPreviousData: true` en opciones | `placeholderData: keepPreviousData` importado de la librería |
| `onSuccess`, `onError` en `useQuery` | Eliminados — usa `useEffect` o los callbacks en `useMutation` |
| `cacheTime` | `gcTime` (garbage collection time) |
| `status === 'loading'` | `status === 'pending'` |

---

## Ejercicios propuestos

1. **`useUsers`** — crea un hook personalizado `useUsers()` que encapsule
   `useQuery` para fetchear usuarios de `jsonplaceholder.typicode.com/users`.
   El hook debe retornar `{ users, isPending, isError }` con tipos explícitos.

2. **`useUpdatePost`** — crea un hook personalizado `useUpdatePost(postId)`
   que encapsule `useMutation` para actualizar un post. En `onSuccess` actualiza
   el caché del post individual con `queryClient.setQueryData`.

3. **Búsqueda con debounce** — combina `useDebounce` de página 9 con `useQuery`.
   El `queryKey` incluye el término debounced. `enabled` es `false` si el término
   tiene menos de 2 caracteres. Muestra un skeleton mientras carga.

---

## Resumen de la página 14

- TanStack Query v5 gestiona automáticamente caché, re-fetching, estados de carga y errores.
- `QueryClientProvider` envuelve toda la app en `main.tsx` con una instancia de `QueryClient`.
- `useQuery` con `queryKey` + `queryFn` — la `queryKey` es la identidad del dato en el caché.
- `isPending` es true solo en la primera carga sin datos en caché. `isFetching` es true en cualquier fetch activo incluyendo re-validaciones.
- `enabled: false` evita el fetch hasta que la condición sea verdadera — útil para queries que dependen de otro dato.
- `useMutation` con `onSuccess` llama a `invalidateQueries` para forzar re-fetch de datos relacionados.
- La actualización optimista en `onMutate` actualiza el caché antes del servidor y hace rollback en `onError`.
- `keepPreviousData` (importado de la librería) evita el parpadeo de "cargando" entre páginas — muestra datos anteriores mientras llegan los nuevos.
- En v5 los callbacks `onSuccess`/`onError` de `useQuery` fueron eliminados — usa `useMutation` o `useEffect` para efectos secundarios.

---

> **Siguiente página →** Proyecto integrador: aplicación completa
> con React Router, Context, TanStack Query, formularios con Zod
> y tests con Vitest.