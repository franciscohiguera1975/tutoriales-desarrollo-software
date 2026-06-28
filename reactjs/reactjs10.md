# Curso React.js + TypeScript — Página 10
## Módulo 3 · Hooks nativos
### `useCallback` — memoización de funciones

---

## ¿Qué es `useCallback`?

`useCallback` **memoriza la referencia de una función** y solo la recrea cuando alguna
de sus dependencias cambia. Sin él, cada render genera una nueva instancia de la función
— aunque el código sea idéntico.

```tsx
const handleClick = useCallback(() => {
  doSomething(id)
}, [id])
// ↑ la misma referencia mientras `id` no cambie
```

```tsx
// ❌ Sin useCallback — nueva función en cada render
const handleDelete = (id: number) => remove(id)   // objeto nuevo → hijo re-renderiza siempre

// ✅ Con useCallback — referencia estable
const handleDelete = useCallback((id: number) => remove(id), [remove])
```

---

## Por qué importa la referencia estable

En JavaScript, dos funciones con el mismo código **no son iguales**:

```tsx
const a = () => {}
const b = () => {}
console.log(a === b) // false — objetos distintos en memoria
```

React.memo compara props con `===`. Si el padre pasa `handleDelete` sin `useCallback`,
el hijo recibe un objeto nuevo en cada render y **siempre re-renderiza**, anulando el memo.

---

## Cuándo usar `useCallback`

| Situación | Usar `useCallback` |
|---|---|
| Función pasada como prop a un hijo con `React.memo` | ✅ Sí |
| Función en el array de dependencias de `useEffect` | ✅ Sí |
| Handler de formulario local sin hijos memoizados | ❌ No |
| Función que no sale del componente | ❌ No |

> **Regla práctica**: si la función no se pasa a ningún hijo ni entra en `useEffect`,
> `useCallback` añade complejidad sin beneficio.

---

## Dependencias de `useCallback`

```tsx
// ✅ Correcto — incluye todos los valores del scope que usa la función
const fn = useCallback(() => {
  doSomethingWith(count, userId)
}, [count, userId])

// ❌ Dependencia olvidada — fn usa count pero no lo declara
const fn = useCallback(() => {
  doSomethingWith(count, userId)
}, [userId])   // count stale cuando cambia
```

---

## PASO 1 — `src/components/MemoizedList.tsx`

Demuestra el problema de re-renders con `React.memo` y cómo `useCallback`
lo soluciona. Un counter independiente provoca renders del padre — sin callback
estable, cada fila re-renderiza aunque sus datos no cambien.

```tsx
// src/components/MemoizedList.tsx

import { useState, useCallback, memo } from 'react'

interface Task {
  id:        number
  text:      string
  completed: boolean
}

const INITIAL_TASKS: Task[] = [
  { id: 1, text: 'Diseñar la interfaz',    completed: false },
  { id: 2, text: 'Implementar los hooks',  completed: true  },
  { id: 3, text: 'Escribir los tests',     completed: false },
  { id: 4, text: 'Revisar accesibilidad',  completed: false },
  { id: 5, text: 'Deploy en producción',   completed: false },
]

// ─── Fila memoizada ──────────────────────────────────────────────────────
let rowRenderCount = 0

const TaskRow = memo(function TaskRow({
  task,
  onToggle,
  onDelete,
}: {
  task:     Task
  onToggle: (id: number) => void
  onDelete: (id: number) => void
}) {
  rowRenderCount++
  const count = rowRenderCount

  return (
    <div style={{
      display:        'flex',
      alignItems:     'center',
      gap:            10,
      padding:        '10px 14px',
      background:     task.completed ? '#f0fdf4' : '#fafafa',
      borderRadius:   8,
      border:         '1px solid',
      borderColor:    task.completed ? '#86efac' : '#e5e5e5',
    }}>
      <input
        type="checkbox"
        checked={task.completed}
        onChange={() => onToggle(task.id)}
        style={{ cursor: 'pointer', width: 16, height: 16 }}
      />
      <span style={{
        flex:           1,
        fontSize:       14,
        textDecoration: task.completed ? 'line-through' : 'none',
        color:          task.completed ? '#666' : '#111',
      }}>
        {task.text}
      </span>
      <span style={{ fontSize: 11, color: '#aaa' }}>render #{count}</span>
      <button
        onClick={() => onDelete(task.id)}
        style={{
          padding:      '2px 8px',
          borderRadius: 4,
          border:       '1px solid #fca5a5',
          background:   '#fef2f2',
          color:        '#dc2626',
          cursor:       'pointer',
          fontSize:     12,
        }}
      >
        ✕
      </button>
    </div>
  )
})

// ─── Padre ───────────────────────────────────────────────────────────────
export default function MemoizedList() {
  const [tasks,   setTasks]   = useState<Task[]>(INITIAL_TASKS)
  const [counter, setCounter] = useState(0)

  // useCallback: onToggle y onDelete tienen referencia estable
  // TaskRow no re-renderiza cuando solo cambia `counter`
  const handleToggle = useCallback((id: number) => {
    setTasks(prev => prev.map(t => t.id === id ? { ...t, completed: !t.completed } : t))
  }, []) // sin dependencias externas — setTasks es estable

  const handleDelete = useCallback((id: number) => {
    setTasks(prev => prev.filter(t => t.id !== id))
  }, [])

  return (
    <div style={{ fontFamily: 'sans-serif', maxWidth: 520, margin: '0 auto', padding: 24 }}>
      <h2 style={{ fontSize: 18, fontWeight: 700, marginBottom: 4 }}>MemoizedList</h2>
      <p style={{ color: '#666', fontSize: 14, marginBottom: 20 }}>
        <code>React.memo</code> + <code>useCallback</code> — las filas no re-renderizan por un counter ajeno.
      </p>

      <div style={{ display: 'flex', gap: 12, alignItems: 'center', marginBottom: 20 }}>
        <button
          onClick={() => setCounter(c => c + 1)}
          style={{ padding: '6px 16px', borderRadius: 6, border: '1px solid #ccc', cursor: 'pointer' }}
        >
          Incrementar counter ({counter})
        </button>
        <span style={{ fontSize: 13, color: '#888' }}>
          ← no debe re-renderizar las filas
        </span>
      </div>

      <div style={{ display: 'flex', flexDirection: 'column', gap: 8 }}>
        {tasks.map(task => (
          <TaskRow
            key={task.id}
            task={task}
            onToggle={handleToggle}
            onDelete={handleDelete}
          />
        ))}
      </div>

      <p style={{ marginTop: 16, fontSize: 12, color: '#aaa' }}>
        Render total de filas: {rowRenderCount}
        {' '}(debería crecer solo al hacer toggle o delete, no al pulsar el counter)
      </p>
    </div>
  )
}
```

### `src/App.tsx`

```tsx
// src/App.tsx

import MemoizedList from './components/MemoizedList'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  MemoizedList    — useCallback + React.memo: evita re-renders    │
// │  2  SearchWithFetch — useCallback en deps de useEffect (sin bucle)  │
// │  3  FilterTable     — tres callbacks estables, tabla memoizada      │
// │  4  PaginatedFetch  — useCallback con [page] para paginación        │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content =
    PASO === 1 ? <MemoizedList /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 720, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

### Prueba esto

- Pulsa "Incrementar counter" varias veces — el número de fila ("render #N") **no debe cambiar**; las filas no re-renderizan
- Marca una tarea como completada — solo **esa fila** incrementa su contador de render
- Elimina una tarea — las demás filas no re-renderizan
- Comenta `useCallback` y sustituye por funciones normales — pulsa el counter y observa que TODAS las filas re-renderizan cada vez
- Nota que `handleToggle` y `handleDelete` tienen deps `[]` — `setTasks` (forma funcional) no necesita declararse

---

## PASO 2 — `src/components/SearchWithFetch.tsx`

`useCallback` para estabilizar una función de fetch que entra en el array de
dependencias de `useEffect`. Sin él, `useEffect` correría en bucle infinito.

```tsx
// src/components/SearchWithFetch.tsx

import { useState, useEffect, useCallback } from 'react'

interface Post {
  id:    number
  title: string
  body:  string
}

export default function SearchWithFetch() {
  const [query,   setQuery]   = useState('')
  const [posts,   setPosts]   = useState<Post[]>([])
  const [loading, setLoading] = useState(false)
  const [error,   setError]   = useState<string | null>(null)

  // useCallback: fetchPosts es estable mientras `query` no cambie.
  // Sin useCallback, sería una función nueva en cada render
  // → useEffect la vería como nueva dep → bucle infinito.
  const fetchPosts = useCallback(async () => {
    setLoading(true)
    setError(null)
    try {
      const url = query.trim()
        ? `https://jsonplaceholder.typicode.com/posts?_limit=5&title_like=${encodeURIComponent(query)}`
        : 'https://jsonplaceholder.typicode.com/posts?_limit=5'
      const res  = await fetch(url)
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      const data: Post[] = await res.json()
      setPosts(data)
    } catch (e) {
      setError(e instanceof Error ? e.message : 'Error desconocido')
    } finally {
      setLoading(false)
    }
  }, [query]) // ← recrear solo cuando query cambia

  // useEffect seguro — fetchPosts es estable, no hay bucle
  useEffect(() => {
    fetchPosts()
  }, [fetchPosts])

  return (
    <div style={{ fontFamily: 'sans-serif', maxWidth: 540, margin: '0 auto', padding: 24 }}>
      <h2 style={{ fontSize: 18, fontWeight: 700, marginBottom: 4 }}>SearchWithFetch</h2>
      <p style={{ color: '#666', fontSize: 14, marginBottom: 20 }}>
        <code>useCallback</code> estabiliza la función de fetch en las deps de <code>useEffect</code>.
      </p>

      <div style={{ display: 'flex', gap: 8, marginBottom: 20 }}>
        <input
          type="text"
          placeholder="Buscar en títulos..."
          value={query}
          onChange={e => setQuery(e.target.value)}
          style={{
            flex:    1,
            padding: '8px 12px',
            border:  '1px solid #ccc',
            borderRadius: 8,
            fontSize: 14,
          }}
        />
        <button
          onClick={fetchPosts}
          style={{
            padding:      '8px 16px',
            borderRadius: 8,
            border:       '1px solid #0070f3',
            background:   '#0070f3',
            color:        'white',
            cursor:       'pointer',
            fontSize:     14,
          }}
        >
          Buscar
        </button>
      </div>

      {error && (
        <div style={{
          padding:      12,
          background:   '#fef2f2',
          border:       '1px solid #fca5a5',
          borderRadius: 8,
          color:        '#dc2626',
          fontSize:     14,
          marginBottom: 16,
        }}>
          {error}
        </div>
      )}

      {loading ? (
        <div style={{ textAlign: 'center', padding: 32, color: '#aaa' }}>Cargando…</div>
      ) : (
        <div style={{ display: 'flex', flexDirection: 'column', gap: 10 }}>
          {posts.map(post => (
            <div key={post.id} style={{
              padding:      '12px 16px',
              background:   '#f9f9f9',
              borderRadius: 8,
              border:       '1px solid #eee',
            }}>
              <div style={{ fontWeight: 600, fontSize: 14, marginBottom: 4 }}>
                {post.title}
              </div>
              <div style={{ fontSize: 13, color: '#666', lineHeight: 1.5 }}>
                {post.body.slice(0, 100)}…
              </div>
            </div>
          ))}
          {posts.length === 0 && !loading && (
            <p style={{ textAlign: 'center', color: '#aaa' }}>Sin resultados.</p>
          )}
        </div>
      )}
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import SearchWithFetch from './components/SearchWithFetch'
```

```tsx
PASO === 2 ? <SearchWithFetch /> :
```

Cambia `PASO = 2` y guarda.

### Prueba esto

- Carga inicial — se llama `fetchPosts` automáticamente via `useEffect`
- Escribe en el buscador — cada cambio en `query` recrea `fetchPosts`, `useEffect` lo detecta y lanza otra búsqueda
- Haz clic en "Buscar" manualmente — llama a `fetchPosts()` directamente (la misma función estable)
- Elimina `useCallback` y define `fetchPosts` como función normal — abre DevTools y observa peticiones en bucle
- Cambia `_limit=5` a `_limit=10` en la URL — la próxima vez que `query` cambie traerá 10 resultados

---

## PASO 3 — `src/components/FilterTable.tsx`

`useCallback` para handlers de fila en una tabla con React.memo.
Múltiples callbacks tipados: editar, eliminar y cambiar estado.

```tsx
// src/components/FilterTable.tsx

import { useState, useCallback, useMemo, memo } from 'react'

interface Employee {
  id:         number
  name:       string
  department: string
  salary:     number
  active:     boolean
}

const EMPLOYEES: Employee[] = [
  { id: 1, name: 'Ana García',    department: 'Diseño',       salary: 2800, active: true  },
  { id: 2, name: 'Luis Pérez',    department: 'Ingeniería',   salary: 3500, active: true  },
  { id: 3, name: 'María López',   department: 'Marketing',    salary: 2400, active: false },
  { id: 4, name: 'Carlos Ruiz',   department: 'Ingeniería',   salary: 4000, active: true  },
  { id: 5, name: 'Sofía Torres',  department: 'Diseño',       salary: 3100, active: true  },
  { id: 6, name: 'Pedro Jiménez', department: 'Marketing',    salary: 2600, active: false },
]

// ─── Fila memoizada ──────────────────────────────────────────────────────
const EmployeeRow = memo(function EmployeeRow({
  emp,
  onToggle,
  onRaise,
  onRemove,
}: {
  emp:      Employee
  onToggle: (id: number) => void
  onRaise:  (id: number, amount: number) => void
  onRemove: (id: number) => void
}) {
  return (
    <tr style={{ opacity: emp.active ? 1 : 0.5 }}>
      <td style={{ padding: '8px 12px', fontWeight: 500 }}>{emp.name}</td>
      <td style={{ padding: '8px 12px', color: '#666' }}>{emp.department}</td>
      <td style={{ padding: '8px 12px', fontWeight: 700 }}>${emp.salary.toLocaleString()}</td>
      <td style={{ padding: '8px 12px' }}>
        <span style={{
          padding:      '2px 10px',
          borderRadius: 999,
          fontSize:     12,
          fontWeight:   600,
          background:   emp.active ? '#dcfce7' : '#f3f4f6',
          color:        emp.active ? '#15803d' : '#6b7280',
        }}>
          {emp.active ? 'Activo' : 'Inactivo'}
        </span>
      </td>
      <td style={{ padding: '8px 12px' }}>
        <div style={{ display: 'flex', gap: 6 }}>
          <button
            onClick={() => onToggle(emp.id)}
            style={{
              padding:      '3px 10px',
              borderRadius: 4,
              border:       '1px solid #d1d5db',
              cursor:       'pointer',
              fontSize:     12,
              background:   'white',
            }}
          >
            {emp.active ? 'Desactivar' : 'Activar'}
          </button>
          <button
            onClick={() => onRaise(emp.id, 200)}
            style={{
              padding:    '3px 10px',
              borderRadius: 4,
              border:     '1px solid #86efac',
              background: '#f0fdf4',
              color:      '#15803d',
              cursor:     'pointer',
              fontSize:   12,
            }}
          >
            +$200
          </button>
          <button
            onClick={() => onRemove(emp.id)}
            style={{
              padding:    '3px 10px',
              borderRadius: 4,
              border:     '1px solid #fca5a5',
              background: '#fef2f2',
              color:      '#dc2626',
              cursor:     'pointer',
              fontSize:   12,
            }}
          >
            ✕
          </button>
        </div>
      </td>
    </tr>
  )
})

// ─── Padre ───────────────────────────────────────────────────────────────
export default function FilterTable() {
  const [employees,   setEmployees]   = useState<Employee[]>(EMPLOYEES)
  const [deptFilter,  setDeptFilter]  = useState('Todos')
  const [showInactive,setShowInactive]= useState(true)

  const departments = useMemo(
    () => ['Todos', ...new Set(EMPLOYEES.map(e => e.department))],
    []
  )

  const visible = useMemo(
    () => employees.filter(e =>
      (deptFilter === 'Todos' || e.department === deptFilter) &&
      (showInactive || e.active)
    ),
    [employees, deptFilter, showInactive]
  )

  // useCallback: handlers estables → EmployeeRow no re-renderiza por filtros
  const handleToggle = useCallback((id: number) => {
    setEmployees(prev => prev.map(e => e.id === id ? { ...e, active: !e.active } : e))
  }, [])

  const handleRaise = useCallback((id: number, amount: number) => {
    setEmployees(prev => prev.map(e => e.id === id ? { ...e, salary: e.salary + amount } : e))
  }, [])

  const handleRemove = useCallback((id: number) => {
    setEmployees(prev => prev.filter(e => e.id !== id))
  }, [])

  const totalSalary = useMemo(() => visible.reduce((s, e) => s + e.salary, 0), [visible])

  return (
    <div style={{ fontFamily: 'sans-serif', maxWidth: 700, margin: '0 auto', padding: 24 }}>
      <h2 style={{ fontSize: 18, fontWeight: 700, marginBottom: 4 }}>FilterTable</h2>
      <p style={{ color: '#666', fontSize: 14, marginBottom: 20 }}>
        Tres callbacks estables para una tabla con <code>React.memo</code>. Cambiar filtros no re-renderiza las filas.
      </p>

      {/* Filtros */}
      <div style={{ display: 'flex', gap: 16, marginBottom: 16, flexWrap: 'wrap' }}>
        <select
          value={deptFilter}
          onChange={e => setDeptFilter(e.target.value)}
          style={{ padding: '6px 10px', border: '1px solid #ccc', borderRadius: 6 }}
        >
          {departments.map(d => <option key={d}>{d}</option>)}
        </select>
        <label style={{ display: 'flex', alignItems: 'center', gap: 6, fontSize: 14, cursor: 'pointer' }}>
          <input
            type="checkbox"
            checked={showInactive}
            onChange={e => setShowInactive(e.target.checked)}
          />
          Mostrar inactivos
        </label>
        <span style={{ fontSize: 13, color: '#888', alignSelf: 'center' }}>
          {visible.length} empleado{visible.length !== 1 ? 's' : ''} · Salario total: ${totalSalary.toLocaleString()}
        </span>
      </div>

      <table style={{ width: '100%', borderCollapse: 'collapse', fontSize: 13 }}>
        <thead>
          <tr style={{ background: '#f5f5f5' }}>
            {['Nombre', 'Depto.', 'Salario', 'Estado', 'Acciones'].map(h => (
              <th key={h} style={{
                textAlign:    'left',
                padding:      '8px 12px',
                borderBottom: '2px solid #e5e5e5',
                fontWeight:   600,
                color:        '#555',
              }}>{h}</th>
            ))}
          </tr>
        </thead>
        <tbody>
          {visible.map(emp => (
            <EmployeeRow
              key={emp.id}
              emp={emp}
              onToggle={handleToggle}
              onRaise={handleRaise}
              onRemove={handleRemove}
            />
          ))}
          {visible.length === 0 && (
            <tr>
              <td colSpan={5} style={{ padding: 24, textAlign: 'center', color: '#aaa' }}>
                Sin empleados para los filtros actuales.
              </td>
            </tr>
          )}
        </tbody>
      </table>
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import FilterTable from './components/FilterTable'
```

```tsx
PASO === 3 ? <FilterTable /> :
```

Cambia `PASO = 3` y guarda.

### Prueba esto

- Cambia el filtro de departamento — las filas visibles cambian pero las filas existentes **no re-renderizan** (observa en DevTools Profiler)
- Haz clic en "+$200" de un empleado — solo esa fila actualiza su salario y re-renderiza
- Haz clic en "Desactivar" — la fila cambia de opacidad y texto, las demás no re-renderizan
- Desmarca "Mostrar inactivos" — las filas inactivas desaparecen sin re-renderizar las activas
- Nota que los tres handlers usan `setEmployees` en forma funcional — no necesitan al propio `employees` en sus deps

---

## PASO 4 — `src/components/PaginatedFetch.tsx`

`useCallback` para una función de fetch con múltiples parámetros.
Al cambiar de página, el callback se recrea con los nuevos parámetros.

```tsx
// src/components/PaginatedFetch.tsx

import { useState, useCallback, useEffect } from 'react'

interface User {
  id:       number
  name:     string
  email:    string
  username: string
}

const PAGE_SIZE = 5

export default function PaginatedFetch() {
  const [page,    setPage]    = useState(1)
  const [users,   setUsers]   = useState<User[]>([])
  const [total,   setTotal]   = useState(0)
  const [loading, setLoading] = useState(false)
  const [error,   setError]   = useState<string | null>(null)

  // useCallback: fetchPage se recrea solo cuando `page` cambia
  const fetchPage = useCallback(async () => {
    setLoading(true)
    setError(null)
    try {
      // JSONPlaceholder: _start y _limit para paginación
      const start = (page - 1) * PAGE_SIZE
      const res   = await fetch(
        `https://jsonplaceholder.typicode.com/users?_start=${start}&_limit=${PAGE_SIZE}`
      )
      if (!res.ok) throw new Error(`HTTP ${res.status}`)
      // El total viene en el header X-Total-Count
      const totalCount = Number(res.headers.get('x-total-count') ?? 10)
      const data: User[] = await res.json()
      setUsers(data)
      setTotal(totalCount)
    } catch (e) {
      setError(e instanceof Error ? e.message : 'Error')
    } finally {
      setLoading(false)
    }
  }, [page]) // ← nueva función solo cuando page cambia

  useEffect(() => { fetchPage() }, [fetchPage])

  const totalPages = Math.ceil(total / PAGE_SIZE) || 1

  return (
    <div style={{ fontFamily: 'sans-serif', maxWidth: 540, margin: '0 auto', padding: 24 }}>
      <h2 style={{ fontSize: 18, fontWeight: 700, marginBottom: 4 }}>PaginatedFetch</h2>
      <p style={{ color: '#666', fontSize: 14, marginBottom: 20 }}>
        <code>useCallback</code> con <code>[page]</code> como dep — fetch al cambiar de página.
      </p>

      {error && (
        <div style={{
          padding:    12,
          background: '#fef2f2',
          border:     '1px solid #fca5a5',
          borderRadius: 8,
          color:      '#dc2626',
          marginBottom: 16,
          fontSize:   14,
        }}>
          {error}
        </div>
      )}

      {loading ? (
        <div style={{ textAlign: 'center', padding: 32, color: '#aaa' }}>Cargando…</div>
      ) : (
        <div style={{ display: 'flex', flexDirection: 'column', gap: 8, marginBottom: 20 }}>
          {users.map(u => (
            <div key={u.id} style={{
              display:        'flex',
              justifyContent: 'space-between',
              alignItems:     'center',
              padding:        '10px 14px',
              background:     '#f9f9f9',
              borderRadius:   8,
              border:         '1px solid #eee',
            }}>
              <div>
                <div style={{ fontWeight: 600, fontSize: 14 }}>{u.name}</div>
                <div style={{ fontSize: 12, color: '#888' }}>@{u.username}</div>
              </div>
              <div style={{ fontSize: 13, color: '#555' }}>{u.email}</div>
            </div>
          ))}
        </div>
      )}

      {/* Paginación */}
      <div style={{ display: 'flex', gap: 8, alignItems: 'center', justifyContent: 'center' }}>
        <button
          onClick={() => setPage(1)}
          disabled={page === 1 || loading}
          style={{ padding: '5px 10px', borderRadius: 6, border: '1px solid #ddd', cursor: 'pointer' }}
        >«</button>
        <button
          onClick={() => setPage(p => Math.max(1, p - 1))}
          disabled={page === 1 || loading}
          style={{ padding: '5px 10px', borderRadius: 6, border: '1px solid #ddd', cursor: 'pointer' }}
        >‹</button>

        {Array.from({ length: totalPages }, (_, i) => i + 1).map(p => (
          <button
            key={p}
            onClick={() => setPage(p)}
            disabled={loading}
            style={{
              padding:    '5px 12px',
              borderRadius: 6,
              border:     '1px solid',
              borderColor: p === page ? '#0070f3' : '#ddd',
              background:  p === page ? '#0070f3' : 'white',
              color:       p === page ? 'white'    : '#333',
              cursor:      'pointer',
              fontWeight:  p === page ? 700 : 400,
            }}
          >
            {p}
          </button>
        ))}

        <button
          onClick={() => setPage(p => Math.min(totalPages, p + 1))}
          disabled={page === totalPages || loading}
          style={{ padding: '5px 10px', borderRadius: 6, border: '1px solid #ddd', cursor: 'pointer' }}
        >›</button>
        <button
          onClick={() => setPage(totalPages)}
          disabled={page === totalPages || loading}
          style={{ padding: '5px 10px', borderRadius: 6, border: '1px solid #ddd', cursor: 'pointer' }}
        >»</button>
      </div>

      <p style={{ textAlign: 'center', fontSize: 12, color: '#aaa', marginTop: 10 }}>
        Página {page} de {totalPages} · {total} usuarios en total
      </p>
    </div>
  )
}
```

### Agrega a `src/App.tsx`

```tsx
import PaginatedFetch from './components/PaginatedFetch'
```

```tsx
PASO === 4 ? <PaginatedFetch /> :
```

Cambia `PASO = 4` y guarda. Desde aquí puedes volver a cualquier paso anterior cambiando la constante.

### Prueba esto

- Haz clic en los botones de paginación — `page` cambia → `fetchPage` se recrea → `useEffect` lo detecta → nueva petición
- Abre DevTools → Network — cada cambio de página genera exactamente una petición `GET /users?_start=N&_limit=5`
- Haz clic en el mismo número de página dos veces — la segunda vez `page` no cambia → `fetchPage` es el mismo → sin petición extra
- Elimina `[page]` de las deps de `useCallback` (deja `[]`) — la paginación deja de funcionar: `fetchPage` siempre usa `page = 1`
- Añade un campo de búsqueda con su propio estado y agrégalo a las deps: `useCallback(async () => { ... }, [page, search])`

---

## Resumen de la página 10

- `useCallback` memoriza la referencia de una función; solo la recrea cuando cambian sus dependencias.
- En JavaScript, funciones con el mismo código tienen referencias distintas — `() => {} === () => {}` es `false`.
- `React.memo` compara props con `===`; una función nueva en cada render siempre activa el re-render del hijo.
- Si una función entra en el array de deps de `useEffect` sin `useCallback`, se genera un bucle infinito.
- Los handlers que usan `setState` en forma funcional (`setState(prev => ...)`) generalmente no necesitan deps.
- Usa la forma funcional del setter para evitar capturar estado stale: `setList(prev => [...prev, item])`.
- `useCallback` sin beneficio añade complejidad — úsalo solo cuando la función sale del componente.

---

> **Siguiente página →** Página 11: hooks personalizados (`useLocalStorage`, `useFetch`, `useDebounce`…).
> **Páginas anteriores →** Página 9: `useMemo` · Página 6: intro a `useRef`, `useCallback`, `useMemo`.
