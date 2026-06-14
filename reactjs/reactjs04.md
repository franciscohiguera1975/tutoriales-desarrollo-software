# Curso React.js + TypeScript — Página 4
## Módulo 2 · Estado y efectos
### `useState` en profundidad: tipos, objetos, arrays y el carrito funcional

---

## ¿Qué es el estado?

En React, el **estado** es cualquier dato que puede cambiar en el tiempo y cuyo cambio
debe reflejarse en la UI. Cuando el estado cambia, React vuelve a renderizar el componente
que lo contiene — y solo ese componente más sus hijos.

La diferencia fundamental con una variable normal:

```tsx
// Variable normal — cambiarla NO actualiza la UI
let contador = 0
function incrementar() {
  contador++ // React no sabe que esto ocurrió
}

// Estado con useState — cambiar el estado SÍ actualiza la UI
import { useState } from 'react'

const [contador, setContador] = useState(0)
function incrementar() {
  setContador(contador + 1) // React re-renderiza el componente
}
```

> **Regla fundamental**: nunca mutes el estado directamente.
> Siempre usa la función setter que retorna `useState`.

---

## Anatomía de `useState`

```tsx
import { useState } from 'react'

const [valor, setValor] = useState(valorInicial)
//     ^        ^                   ^
//     estado   setter              valor de inicio (solo se usa en el primer render)
```

`useState` retorna un array de exactamente dos elementos:
- **`valor`** — el valor actual del estado. Solo lectura.
- **`setValor`** — función para actualizar el estado. Llamarla dispara un re-render.

---

## Inferencia de tipos vs tipos explícitos

TypeScript infiere el tipo del estado a partir del valor inicial.
En la mayoría de casos no necesitas anotar el tipo manualmente:

```tsx
import { useState } from 'react'

// TypeScript infiere: useState<number>
const [count, setCount] = useState(0)

// TypeScript infiere: useState<string>
const [name, setName] = useState('')

// TypeScript infiere: useState<boolean>
const [isOpen, setIsOpen] = useState(false)
```

### Cuándo sí necesitas el tipo explícito

Hay tres situaciones donde la inferencia no alcanza y debes anotarlo:

```tsx
import { useState } from 'react'

// 1. Estado que empieza en null pero luego tendrá un tipo concreto
const [user, setUser] = useState<User | null>(null)

// 2. Arrays vacíos — TypeScript inferiría never[] sin la anotación
const [items, setItems] = useState<string[]>([])

// 3. Unión de tipos que el valor inicial no representa completamente
type Status = 'idle' | 'loading' | 'success' | 'error'
const [status, setStatus] = useState<Status>('idle')
// Sin la anotación, TypeScript inferiría el tipo literal 'idle'
// y no aceptaría setStatus('loading')
```

---

## Estado con tipos primitivos

```tsx
// src/components/DigitalCounter.tsx

import { useState } from 'react'

interface DigitalCounterProps {
  initialValue?: number
  step?: number
  label?: string
}

export default function DigitalCounter({
  initialValue = 0,
  step = 1,
  label = 'Contador',
}) {
  const [count, setCount] = useState(initialValue)

  function increment() {
    setCount(count + step)
  }

  function decrement() {
    setCount(count - step)
  }

  function reset() {
    setCount(initialValue)
  }

  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: 12 }}>
      <span style={{ fontSize: 14, color: '#666' }}>{label}</span>
      <button onClick={decrement} style={btnStyle}>−</button>
      <span style={{ fontSize: 20, fontWeight: 600, minWidth: 40, textAlign: 'center' }}>
        {count}
      </span>
      <button onClick={increment} style={btnStyle}>+</button>
      <button onClick={reset} style={{ ...btnStyle, fontSize: 12, color: '#999' }}>
        Reset
      </button>
    </div>
  )
}

const btnStyle = {
  width: 32,
  height: 32,
  borderRadius: 6,
  border: '1px solid #ddd',
  background: '#f5f5f5',
  cursor: 'pointer',
  fontSize: 16,
}
```

### Prueba esto

- Pasa `step={5}` — los botones `+` y `−` incrementan de 5 en 5; cambia a `step={0.5}` para valores decimales
- Pasa `initialValue={10}` — el contador empieza en 10 y el botón `Reset` vuelve a 10, no a 0
- Pasa `label="Temperatura (°C)"` — el texto de la etiqueta cambia sin tocar el componente
- Cambia `setCount(count + step)` en `increment` a `setCount((prev) => prev + step)` — mismo resultado con un solo clic, pero la forma funcional es más segura cuando hay múltiples updates seguidos
- Añade `min?: number` y `max?: number` a las props y deshabilita los botones con `disabled={count <= min}` y `disabled={count >= max}`
- Cambia `minWidth: 40` a `60` — el número tiene más espacio; evita que el layout cambie de ancho con números de 2 o 3 cifras

---

## Actualización basada en el estado previo

Cuando el nuevo estado depende del estado anterior, **usa siempre la forma funcional del setter**.
Esto garantiza que trabajas con el valor más reciente, incluso si React agrupa varios updates:

```tsx
import { useState } from 'react'

export default function SafeCounter() {
  const [count, setCount] = useState(0)

  function increment() {
    // ❌ Puede fallar si React agrupa renders
    setCount(count + 1)

    // ✅ Siempre correcto — prev es garantizado el valor actual
    setCount((prev) => prev + 1)
  }

  // Ejemplo donde la diferencia importa: incrementar 3 veces seguidas
  function incrementThree() {
    // ❌ Las tres líneas leen el mismo valor de count — resultado: +1
    setCount(count + 1)
    setCount(count + 1)
    setCount(count + 1)

    // ✅ Cada llamada recibe el prev actualizado — resultado: +3
    setCount((prev) => prev + 1)
    setCount((prev) => prev + 1)
    setCount((prev) => prev + 1)
  }

  return (
    <div>
      <p>Contador: {count}</p>
      <button onClick={increment}>+1</button>
      <button onClick={incrementThree}>+3</button>
    </div>
  )
}
```

> Usa `setValor(prev => ...)` siempre que el nuevo valor dependa del anterior.
> Es un hábito que evita bugs difíciles de reproducir.

### Prueba esto

- Haz clic en `+3` — el contador sube exactamente 3; cambia las 3 líneas funcionales a `setCount(count + 1)` y observa que solo sube 1 (todas leen el mismo `count`)
- Añade un cuarto botón `+10` que llame a `setCount((prev) => prev + 1)` en un bucle 10 veces — cada llamada recibe el `prev` actualizado por la anterior
- Agrega `console.log('render', count)` al inicio del componente — observa cuántas veces renderiza React al hacer clic en `+3`
- Cambia `setCount((prev) => prev + 1)` a `setCount((prev) => prev * 2)` en `incrementThree` — el doble se aplica 3 veces en cadena
- Añade un botón `−1` con `setCount((prev) => prev - 1)` para verificar que la forma funcional también funciona con decrementos

---

## Estado con objetos

Cuando el estado es un objeto, **nunca lo mutes directamente**.
Debes crear un objeto nuevo con los campos actualizados usando spread (`...`):

```tsx
import { useState } from 'react'

interface UserProfile {
  name: string
  email: string
  age: number
}

export default function UserProfileForm() {
  const [profile, setProfile] = useState<UserProfile>({
    name: '',
    email: '',
    age: 0,
  })

  function handleChange(field: keyof UserProfile, value: string | number) {
    setProfile((prev) => ({
      ...prev,        // copia todos los campos actuales
      [field]: value, // sobreescribe solo el campo que cambió
    }))
  }

  return (
    <form style={{ display: 'flex', flexDirection: 'column', gap: 10, maxWidth: 320 }}>
      <input
        placeholder="Nombre"
        value={profile.name}
        onChange={(e) => handleChange('name', e.target.value)}
        style={inputStyle}
      />
      <input
        placeholder="Email"
        type="email"
        value={profile.email}
        onChange={(e) => handleChange('email', e.target.value)}
        style={inputStyle}
      />
      <input
        placeholder="Edad"
        type="number"
        value={profile.age}
        onChange={(e) => handleChange('age', Number(e.target.value))}
        style={inputStyle}
      />

      <div style={{ marginTop: 8, padding: 12, background: '#f5f5f5', borderRadius: 6 }}>
        <p style={{ margin: 0, fontSize: 13 }}>
          <strong>{profile.name || '—'}</strong> · {profile.email || '—'} · {profile.age || '—'} años
        </p>
      </div>
    </form>
  )
}

const inputStyle = {
  padding: '8px 12px',
  border: '1px solid #ddd',
  borderRadius: 6,
  fontSize: 14,
}
```

```tsx
// ❌ Mutación directa — React NO detecta el cambio y NO re-renderiza
profile.name = 'Ana'
setProfile(profile) // mismo objeto en memoria = React lo ignora

// ✅ Nuevo objeto — React detecta el cambio y re-renderiza
setProfile({ ...profile, name: 'Ana' })
```

### Prueba esto

- Escribe en el campo Nombre — el preview debajo se actualiza en tiempo real porque cada keystroke dispara un re-render con el nuevo estado
- Cambia `{ ...prev, [field]: value }` a `{ ...prev, name: 'fijo' }` — el nombre siempre queda en `'fijo'` aunque escribas en el input; experimenta con otros campos fijos para entender el spread
- Añade `bio: string` a `UserProfile` con valor inicial `''` y agrega un `<textarea>` que llame `handleChange('bio', e.target.value)` — la misma función maneja todos los campos
- Añade un botón "Limpiar" que llame `setProfile({ name: '', email: '', age: 0 })` — muestra cómo resetear todo el objeto en una sola operación
- Cambia `profile.name = 'Ana'; setProfile(profile)` (mutación directa) — observa que React NO re-renderiza porque el objeto en memoria es el mismo
- Añade `required` a los inputs y un botón "Guardar" con un handler que solo haga `alert` si todos los campos están llenos

---

## Estado con arrays

Con arrays la regla es la misma: **nunca mutes, siempre crea uno nuevo**.
Las operaciones más comunes:

```tsx
import { useState } from 'react'

interface Task {
  id: number
  text: string
  done: boolean
}

export default function TaskManager() {
  const [tasks, setTasks] = useState<Task[]>([])
  const [input, setInput] = useState('')

  // AGREGAR — spread del array anterior más el nuevo item
  function addTask() {
    if (!input.trim()) return
    setTasks((prev) => [
      ...prev,
      { id: Date.now(), text: input.trim(), done: false },
    ])
    setInput('')
  }

  // ELIMINAR — filter crea un nuevo array sin el elemento
  function removeTask(id: number) {
    setTasks((prev) => prev.filter((task) => task.id !== id))
  }

  // ACTUALIZAR — map crea un nuevo array con el elemento modificado
  function toggleTask(id: number) {
    setTasks((prev) =>
      prev.map((task) =>
        task.id === id ? { ...task, done: !task.done } : task
      )
    )
  }

  return (
    <div style={{ maxWidth: 380 }}>
      <div style={{ display: 'flex', gap: 8, marginBottom: 16 }}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && addTask()}
          placeholder="Nueva tarea..."
          style={{ flex: 1, padding: '8px 12px', borderRadius: 6, border: '1px solid #ddd' }}
        />
        <button
          onClick={addTask}
          style={{ padding: '8px 16px', background: '#0070f3', color: '#fff', border: 'none', borderRadius: 6, cursor: 'pointer' }}
        >
          Agregar
        </button>
      </div>

      {tasks.length === 0 && (
        <p style={{ color: '#999', fontSize: 14 }}>No hay tareas. ¡Agrega una!</p>
      )}

      <ul style={{ listStyle: 'none', padding: 0 }}>
        {tasks.map((task) => (
          <li
            key={task.id}
            style={{
              display: 'flex',
              alignItems: 'center',
              gap: 10,
              padding: '10px 0',
              borderBottom: '1px solid #eee',
            }}
          >
            <input
              type="checkbox"
              checked={task.done}
              onChange={() => toggleTask(task.id)}
            />
            <span
              style={{
                flex: 1,
                textDecoration: task.done ? 'line-through' : 'none',
                color: task.done ? '#aaa' : '#333',
              }}
            >
              {task.text}
            </span>
            <button
              onClick={() => removeTask(task.id)}
              style={{ background: 'none', border: 'none', cursor: 'pointer', color: '#e00', fontSize: 16 }}
            >
              ✕
            </button>
          </li>
        ))}
      </ul>

      {tasks.length > 0 && (
        <p style={{ fontSize: 13, color: '#888', marginTop: 8 }}>
          {tasks.filter((t) => t.done).length} de {tasks.length} completadas
        </p>
      )}
    </div>
  )
}
```

### Operaciones con arrays — resumen

| Operación | Método correcto | Método incorrecto |
|---|---|---|
| Agregar | `[...prev, nuevoItem]` | `prev.push(item)` |
| Eliminar | `prev.filter(x => x.id !== id)` | `prev.splice(index, 1)` |
| Actualizar | `prev.map(x => x.id === id ? {...x, cambio} : x)` | `prev[i].campo = valor` |
| Reordenar | `[...prev].sort(...)` | `prev.sort(...)` |

> `push`, `splice` y `sort` **mutan** el array original. Con estado de React siempre
> usa sus equivalentes inmutables.

### Prueba esto

- Agrega dos tareas, marca una, luego desmárcala — el estado de `done` se alterna con cada clic gracias al `!task.done` en el `map`
- Cambia `setTasks((prev) => prev.filter(...))` en `removeTask` a `tasks.splice(0, 1); setTasks(tasks)` — observa que la UI NO se actualiza; React no detecta la mutación del mismo array
- Añade una función `clearCompleted()` que use `prev.filter((t) => !t.done)` y un botón "Limpiar completadas" que solo aparezca si hay alguna tarea con `done: true`
- Añade `priority?: 'alta' | 'normal'` a la interfaz `Task` y un `<select>` en el formulario para elegirla; muéstralo como un badge de color en la lista
- Cambia `{ id: Date.now(), ... }` a un contador incremental `useRef` para ids más predecibles — aunque `Date.now()` es suficiente para este caso
- Añade un estado `const [filter, setFilter] = useState<'all' | 'done' | 'pending'>('all')` y botones para filtrar las tareas visibles sin eliminarlas del estado

---

## El carrito de la página 3 — ahora funcional

En la página 3 dejamos el carrito con arrays estáticos y comentarios
`// Página 4: ...`. Aquí lo completamos con `useState`.

---

### `src/components/CatalogProductItem.tsx`

```tsx
// src/components/CatalogProductItem.tsx
// Sin cambios respecto a la página 3 — el componente ya era correcto

interface CatalogProductItemProps {
  id: number
  name: string
  price: number
  onAddToCart: (id: number, name: string, price: number) => void
}

export default function CatalogProductItem({
  id,
  name,
  price,
  onAddToCart,
}: CatalogProductItemProps) {
  return (
    <div
      style={{
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
        padding: '12px 0',
        borderBottom: '1px solid #eee',
      }}
    >
      <div>
        <p style={{ margin: 0, fontWeight: 500 }}>{name}</p>
        <p style={{ margin: 0, fontSize: 13, color: '#888' }}>${price.toFixed(2)}</p>
      </div>
      <button
        onClick={() => onAddToCart(id, name, price)}
        style={{
          backgroundColor: '#0070f3',
          color: '#fff',
          border: 'none',
          borderRadius: 6,
          padding: '6px 14px',
          cursor: 'pointer',
        }}
      >
        + Agregar
      </button>
    </div>
  )
}
```

---

### `src/components/ShoppingCartSummary.tsx`

```tsx
// src/components/ShoppingCartSummary.tsx
// Sin cambios respecto a la página 3

interface CartItem {
  id: number
  name: string
  price: number
}

interface ShoppingCartSummaryProps {
  items: CartItem[]
  onClearCart: () => void
}

export default function ShoppingCartSummary({
  items,
  onClearCart,
}: ShoppingCartSummaryProps) {
  const total = items.reduce((acc, item) => acc + item.price, 0)

  return (
    <div
      style={{
        border: '1px solid #ddd',
        borderRadius: 10,
        padding: 16,
        marginTop: 24,
      }}
    >
      <h3 style={{ marginTop: 0 }}>Carrito ({items.length} items)</h3>

      {items.length === 0 && (
        <p style={{ color: '#999' }}>El carrito está vacío.</p>
      )}

      <ul style={{ listStyle: 'none', padding: 0 }}>
        {items.map((item) => (
          <li
            key={item.id}
            style={{ display: 'flex', justifyContent: 'space-between', padding: '6px 0' }}
          >
            <span>{item.name}</span>
            <span>${item.price.toFixed(2)}</span>
          </li>
        ))}
      </ul>

      {items.length > 0 && (
        <>
          <hr />
          <div style={{ display: 'flex', justifyContent: 'space-between', fontWeight: 600 }}>
            <span>Total</span>
            <span>${total.toFixed(2)}</span>
          </div>
          <button
            onClick={onClearCart}
            style={{
              marginTop: 12,
              backgroundColor: '#e00',
              color: '#fff',
              border: 'none',
              borderRadius: 6,
              padding: '8px 16px',
              cursor: 'pointer',
              width: '100%',
            }}
          >
            Vaciar carrito
          </button>
        </>
      )}
    </div>
  )
}
```

---

### `src/App.tsx` — navegador de pasos

```tsx
// src/App.tsx

import { useState } from 'react'
import DigitalCounter      from './components/DigitalCounter'
import SafeCounter         from './components/SafeCounter'
import UserProfileForm     from './components/UserProfileForm'
import TaskManager         from './components/TaskManager'
import CatalogProductItem  from './components/CatalogProductItem'
import ShoppingCartSummary from './components/ShoppingCartSummary'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  DigitalCounter    — estado numérico con step y reset            │
// │  2  SafeCounter       — forma funcional prev => prev + 1            │
// │  3  UserProfileForm   — estado con objeto + spread update           │
// │  4  TaskManager       — estado con array: filter, map, spread       │
// │  5  Carrito useState  — array de objetos + lógica en App.tsx        │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

interface CartItem { id: number; name: string; price: number }

const catalog = [
  { id: 1, name: 'Teclado mecánico',  price: 89.99 },
  { id: 2, name: 'Monitor 27"',       price: 349.99 },
  { id: 3, name: 'Mouse inalámbrico', price: 29.99 },
]

export default function App() {
  const [cartItems, setCartItems] = useState<CartItem[]>([])

  function handleAddToCart(id: number, name: string, price: number) {
    const alreadyInCart = cartItems.some((item) => item.id === id)
    if (alreadyInCart) return
    setCartItems((prev) => [...prev, { id, name, price }])
  }

  function handleClearCart() {
    setCartItems([])
  }

  const content =
    PASO === 1 ? <DigitalCounter label="Contador" step={1} /> :
    PASO === 2 ? <SafeCounter /> :
    PASO === 3 ? <UserProfileForm /> :
    PASO === 4 ? <TaskManager /> :
    PASO === 5 ? (
      <>
        <h1 style={{ fontSize: 22 }}>Tienda</h1>
        <section>
          {catalog.map((p) => (
            <CatalogProductItem
              key={p.id}
              id={p.id}
              name={p.name}
              price={p.price}
              onAddToCart={handleAddToCart}
            />
          ))}
        </section>
        <ShoppingCartSummary items={cartItems} onClearCart={handleClearCart} />
      </>
    ) :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 480, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

### Prueba esto

- Navega al PASO 5 y agrega productos al carrito — haz clic en "Agregar" varias veces en el mismo producto; solo aparece una vez por el check `alreadyInCart`
- Haz clic en "Vaciar carrito" — `setCartItems([])` limpia el array y el total desaparece
- Elimina el check `alreadyInCart` — ahora el mismo producto puede agregarse varias veces y el total se acumula
- Observa que `CatalogProductItem` y `ShoppingCartSummary` son idénticos a la página 3 — el único cambio fue reemplazar el array estático por `useState`
- Añade `quantity: number` al `CartItem` y en lugar de bloquear duplicados, usa `setCartItems((prev) => prev.map((item) => item.id === id ? { ...item, quantity: item.quantity + 1 } : item))`

---

## Múltiples estados vs un objeto de estado

Una pregunta frecuente: ¿varios `useState` separados o un solo objeto?

```tsx
// Opción A — estados separados (preferida para valores independientes)
const [name, setName]       = useState('')
const [email, setEmail]     = useState('')
const [isLoading, setIsLoading] = useState(false)
// Cada uno se actualiza de forma independiente — claro y predecible

// Opción B — objeto de estado (mejor cuando los campos siempre cambian juntos)
interface FormState {
  name: string
  email: string
  isLoading: boolean
}
const [form, setForm] = useState<FormState>({
  name: '',
  email: '',
  isLoading: false,
})
// Actualizar un campo requiere spread:
setForm((prev) => ({ ...prev, name: 'Ana' }))
```

**Regla práctica**: si dos valores *siempre* se actualizan juntos, ponlos en un objeto.
Si se actualizan por separado en distintos momentos, usa `useState` separados.

---

## Resumen de la página 4

- El estado es datos que cambian en el tiempo y cuyo cambio dispara un re-render.
- `useState` retorna `[valor, setter]` — nunca mutes el valor directamente.
- TypeScript infiere el tipo del estado desde el valor inicial. Solo anota explícitamente cuando el estado empieza en `null`, es un array vacío o es una unión de literales.
- La forma funcional `setValor(prev => ...)` es obligatoria cuando el nuevo valor depende del anterior.
- Con objetos: `{ ...prev, campo: nuevoValor }` — nunca `objeto.campo = valor`.
- Con arrays: `filter`, `map` y spread — nunca `push`, `splice` ni `sort` directamente.
- El estado vive arriba en el árbol y baja como props — los componentes hijos no necesitan cambiar cuando el padre adopta `useState`.

---

> **Siguiente página →** `useEffect`: cuándo y cómo usarlo, dependencias,
> limpieza de efectos y los errores más comunes.