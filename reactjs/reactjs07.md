# Curso React.js + TypeScript — Página 7
## Módulo 3 · Hooks nativos
### `useReducer`: estado complejo con acciones tipadas

---

## ¿Qué es `useReducer`?

`useReducer` es una alternativa a `useState` para gestionar estado complejo.
En lugar de llamar a un setter con el nuevo valor, describes **qué ocurrió**
mediante una **acción**, y una función **reducer** decide cómo cambia el estado.

```tsx
const [state, dispatch] = useReducer(reducer, initialState)
//     ^        ^                    ^         ^
//     estado   disparar acción      función    valor inicial
```

El reducer es una función pura que recibe el estado actual y una acción,
y retorna el nuevo estado:

```ts
function reducer(state: State, action: Action): State {
  // nunca muta state — siempre retorna un objeto nuevo
}
```

---

## `useState` vs `useReducer`

```tsx
// useState — simple, para valores independientes
const [name,    setName]    = useState('')
const [email,   setEmail]   = useState('')
const [loading, setLoading] = useState(false)
const [error,   setError]   = useState<string | null>(null)

// Para enviar el formulario necesitas coordinar 3 setters
setLoading(true)
setError(null)
// ... await fetch
setLoading(false)
setError('Error al enviar')

// useReducer — cuando el estado tiene múltiples campos relacionados
// y las transiciones deben ser consistentes
dispatch({ type: 'SUBMIT_START' })   // loading=true, error=null
dispatch({ type: 'SUBMIT_ERROR', message: 'Error al enviar' })  // loading=false, error=mensaje
```

| Usa `useState` cuando... | Usa `useReducer` cuando... |
|---|---|
| El estado es un valor simple | El estado es un objeto con varios campos |
| Las actualizaciones son independientes | Varias partes del estado cambian juntas |
| La lógica es corta y directa | La lógica de transición es compleja |
| 1–2 valores relacionados | Múltiples acciones sobre el mismo estado |

---

## Tipar acciones con unión discriminada

La clave del tipado de `useReducer` en TypeScript es la **unión discriminada**:
cada acción es un objeto con un campo `type` literal único. TypeScript usa ese
campo para saber exactamente qué forma tiene cada acción dentro del `switch`.

```ts
// Unión discriminada — cada rama del switch tiene su tipo exacto
type CounterAction =
  | { type: 'INCREMENT' }           // sin payload
  | { type: 'DECREMENT' }           // sin payload
  | { type: 'RESET' }               // sin payload
  | { type: 'SET'; payload: number } // con payload tipado

function reducer(state: State, action: CounterAction): State {
  switch (action.type) {
    case 'SET':
      // TypeScript sabe que action.payload es number aquí
      return { count: action.payload }
    case 'INCREMENT':
      // TypeScript sabe que NO hay payload aquí
      return { count: state.count + 1 }
  }
}
```

> El campo `type` con un string literal es la convención universal.
> TypeScript lo usa para estrechar el tipo dentro de cada `case`.

---

## PASO 1 — `src/components/BasicCounter.tsx`

El ejemplo más simple: `useReducer` con 4 acciones, una de ellas con `payload`.

```tsx
// src/components/BasicCounter.tsx

import { useReducer } from 'react'

type CounterAction =
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'RESET' }
  | { type: 'SET'; payload: number }

interface CounterState {
  count: number
}

function counterReducer(
  state: CounterState,
  action: CounterAction
): CounterState {
  switch (action.type) {
    case 'INCREMENT': return { count: state.count + 1 }
    case 'DECREMENT': return { count: state.count - 1 }
    case 'RESET':     return { count: 0 }
    case 'SET':       return { count: action.payload }
  }
}

const INITIAL_STATE: CounterState = { count: 0 }

export default function BasicCounter() {
  const [state, dispatch] = useReducer(counterReducer, INITIAL_STATE)

  return (
    <div style={{ display: 'flex', flexDirection: 'column', gap: 12, maxWidth: 200 }}>
      <p style={{ fontFamily: 'monospace', fontSize: 32, margin: 0, textAlign: 'center' }}>
        {state.count}
      </p>
      <div style={{ display: 'flex', gap: 8, justifyContent: 'center' }}>
        <button
          onClick={() => dispatch({ type: 'DECREMENT' })}
          style={btnStyle}
        >
          −
        </button>
        <button
          onClick={() => dispatch({ type: 'INCREMENT' })}
          style={btnStyle}
        >
          +
        </button>
      </div>
      <button
        onClick={() => dispatch({ type: 'SET', payload: 100 })}
        style={{ ...btnStyle, fontSize: 12 }}
      >
        Poner en 100
      </button>
      <button
        onClick={() => dispatch({ type: 'RESET' })}
        style={{ ...btnStyle, background: '#f3f4f6', color: '#6b7280' }}
      >
        Reset
      </button>
    </div>
  )
}

const btnStyle: React.CSSProperties = {
  padding: '8px 16px',
  border: 'none',
  borderRadius: 6,
  background: '#0070f3',
  color: '#fff',
  cursor: 'pointer',
  fontWeight: 500,
}
```

### `src/App.tsx`

```tsx
// src/App.tsx

import BasicCounter from './components/BasicCounter'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  BasicCounter      — useReducer básico con acciones tipadas      │
// │  2  RegistrationForm  — formulario con validación y estados de envío│
// │  3  ShoppingCart      — carrito de compras completo                 │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content =
    PASO === 1 ? <BasicCounter /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 600, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

### Prueba esto

- Pulsa `+` varias veces — el contador sube de uno en uno; observa que cada dispatch pasa por el reducer antes de actualizar la vista
- Pulsa `−` cuando el contador está en 0 — baja a negativos sin restricción; añade `case 'DECREMENT': return { count: Math.max(0, state.count - 1) }` para impedirlo
- Pulsa "Poner en 100" — el contador salta directamente a 100 usando la acción `SET` con `payload: 100`; cambia el payload a `payload: 42` y guarda para ver el efecto
- Pulsa "Reset" desde cualquier valor — vuelve siempre a 0 porque el reducer retorna `{ count: 0 }`, no el `INITIAL_STATE`; comprueba que es lo mismo en este caso
- Cambia `INITIAL_STATE` a `{ count: 10 }` — el contador arranca en 10 pero "Reset" sigue yendo a 0; si quieres que Reset vuelva al valor inicial, cambia `case 'RESET': return INITIAL_STATE`
- Añade una acción `| { type: 'DOUBLE' }` al tipo y `case 'DOUBLE': return { count: state.count * 2 }` al reducer — TypeScript avisa si olvidas el case; despacha con un nuevo botón

---

## PASO 2 — `src/components/RegistrationForm.tsx`

Formulario con validación y estados de envío. Un único `useReducer` gestiona
campos, errores y status del envío — todo consistente en cada transición.

```tsx
// src/components/RegistrationForm.tsx

import { useReducer } from 'react'

interface FormState {
  name:     string
  email:    string
  password: string
  errors:   Partial<Record<'name' | 'email' | 'password', string>>
  status:   'idle' | 'submitting' | 'success' | 'error'
}

type FormAction =
  | { type: 'SET_FIELD'; field: keyof Pick<FormState, 'name' | 'email' | 'password'>; value: string }
  | { type: 'SET_ERRORS'; errors: FormState['errors'] }
  | { type: 'SUBMIT_START' }
  | { type: 'SUBMIT_SUCCESS' }
  | { type: 'SUBMIT_ERROR' }
  | { type: 'RESET' }

const INITIAL_STATE: FormState = {
  name:     '',
  email:    '',
  password: '',
  errors:   {},
  status:   'idle',
}

function formReducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case 'SET_FIELD':
      return {
        ...state,
        [action.field]: action.value,
        // Limpia el error del campo al escribir
        errors: { ...state.errors, [action.field]: undefined },
      }
    case 'SET_ERRORS':
      return { ...state, errors: action.errors }
    case 'SUBMIT_START':
      return { ...state, status: 'submitting' }
    case 'SUBMIT_SUCCESS':
      return { ...INITIAL_STATE, status: 'success' }
    case 'SUBMIT_ERROR':
      return { ...state, status: 'error' }
    case 'RESET':
      return INITIAL_STATE
  }
}

export default function RegistrationForm() {
  const [state, dispatch] = useReducer(formReducer, INITIAL_STATE)

  function validate(): boolean {
    const errors: FormState['errors'] = {}
    if (!state.name.trim())         errors.name     = 'El nombre es requerido'
    if (!state.email.includes('@')) errors.email    = 'Email inválido'
    if (state.password.length < 6)  errors.password = 'Mínimo 6 caracteres'

    if (Object.keys(errors).length > 0) {
      dispatch({ type: 'SET_ERRORS', errors })
      return false
    }
    return true
  }

  async function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    if (!validate()) return

    dispatch({ type: 'SUBMIT_START' })
    // Simulación de llamada a API
    await new Promise((resolve) => setTimeout(resolve, 1200))
    dispatch({ type: 'SUBMIT_SUCCESS' })
  }

  const isSubmitting = state.status === 'submitting'

  return (
    <form
      onSubmit={handleSubmit}
      style={{ display: 'flex', flexDirection: 'column', gap: 12, maxWidth: 320 }}
    >
      {state.status === 'success' && (
        <div style={{ padding: 12, background: '#dcfce7', borderRadius: 6, color: '#166534' }}>
          ✅ Registro exitoso
        </div>
      )}

      <div>
        <input
          value={state.name}
          onChange={(e) =>
            dispatch({ type: 'SET_FIELD', field: 'name', value: e.target.value })
          }
          placeholder="Nombre completo"
          disabled={isSubmitting}
          style={inputStyle(!!state.errors.name)}
        />
        {state.errors.name && (
          <p style={errorStyle}>{state.errors.name}</p>
        )}
      </div>

      <div>
        <input
          type="email"
          value={state.email}
          onChange={(e) =>
            dispatch({ type: 'SET_FIELD', field: 'email', value: e.target.value })
          }
          placeholder="Correo electrónico"
          disabled={isSubmitting}
          style={inputStyle(!!state.errors.email)}
        />
        {state.errors.email && (
          <p style={errorStyle}>{state.errors.email}</p>
        )}
      </div>

      <div>
        <input
          type="password"
          value={state.password}
          onChange={(e) =>
            dispatch({ type: 'SET_FIELD', field: 'password', value: e.target.value })
          }
          placeholder="Contraseña (mín. 6 caracteres)"
          disabled={isSubmitting}
          style={inputStyle(!!state.errors.password)}
        />
        {state.errors.password && (
          <p style={errorStyle}>{state.errors.password}</p>
        )}
      </div>

      <div style={{ display: 'flex', gap: 8 }}>
        <button
          type="submit"
          disabled={isSubmitting}
          style={{
            flex: 1, padding: '10px',
            background: isSubmitting ? '#93c5fd' : '#0070f3',
            color: '#fff', border: 'none', borderRadius: 6,
            cursor: isSubmitting ? 'not-allowed' : 'pointer',
            fontWeight: 500,
          }}
        >
          {isSubmitting ? 'Registrando...' : 'Registrar'}
        </button>
        <button
          type="button"
          onClick={() => dispatch({ type: 'RESET' })}
          disabled={isSubmitting}
          style={{
            padding: '10px 16px',
            background: '#f3f4f6', color: '#6b7280',
            border: 'none', borderRadius: 6, cursor: 'pointer',
          }}
        >
          Limpiar
        </button>
      </div>
    </form>
  )
}

function inputStyle(hasError: boolean): React.CSSProperties {
  return {
    width: '100%',
    padding: '8px 12px',
    border: `1px solid ${hasError ? '#ef4444' : '#d1d5db'}`,
    borderRadius: 6,
    fontSize: 14,
    boxSizing: 'border-box',
  }
}

const errorStyle: React.CSSProperties = {
  margin: '4px 0 0',
  fontSize: 12,
  color: '#ef4444',
}
```

### Prueba esto

- Envía el formulario con todos los campos vacíos — aparecen tres mensajes de error simultáneamente; nota que una sola acción `SET_ERRORS` actualiza todos los errores de golpe en lugar de tres llamadas a setter
- Empieza a escribir en el campo Nombre después de ver el error — el error desaparece de inmediato porque `SET_FIELD` hace `errors: { ...state.errors, [action.field]: undefined }`
- Envía con datos válidos y observa el estado `submitting` — el botón cambia de texto y el color a azul claro; los inputs se deshabilitan durante la simulación de 1,2 s
- Envía el formulario correctamente — aparece el banner verde y los campos se vacían; esto ocurre porque `SUBMIT_SUCCESS` retorna `{ ...INITIAL_STATE, status: 'success' }` en lugar del estado actual
- Cambia el email de validación de `!state.email.includes('@')` a una regex más estricta — observa cómo la lógica de validación queda solo en la función `validate` sin afectar el reducer
- Añade la acción `| { type: 'SUBMIT_ERROR' }` al flujo haciendo que `handleSubmit` despache `SUBMIT_ERROR` si el email termina en `@fail.com`; observa el estado `error` en la UI

### Agrega a `src/App.tsx`

```tsx
import RegistrationForm from './components/RegistrationForm'
```

```tsx
PASO === 2 ? <RegistrationForm /> :
```

Cambia `PASO = 2` y guarda.

---

## `src/components/ShoppingCart.tsx`

El carrito del curso — ahora con `useReducer`. Todo el estado del carrito
vive en un único reducer: items, visibilidad del panel y todas las operaciones.

```tsx
// src/components/ShoppingCart.tsx

import { useReducer, useMemo } from 'react'

interface CartItem {
  id:       number
  name:     string
  price:    number
  quantity: number
}

interface CartState {
  items:  CartItem[]
  isOpen: boolean
}

type CartAction =
  | { type: 'ADD_ITEM';    item: Omit<CartItem, 'quantity'> }
  | { type: 'REMOVE_ITEM'; id: number }
  | { type: 'INCREMENT';   id: number }
  | { type: 'DECREMENT';   id: number }
  | { type: 'CLEAR' }
  | { type: 'TOGGLE_CART' }

function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM': {
      const exists = state.items.find((i) => i.id === action.item.id)
      if (exists) {
        return {
          ...state,
          items: state.items.map((i) =>
            i.id === action.item.id ? { ...i, quantity: i.quantity + 1 } : i
          ),
        }
      }
      return {
        ...state,
        items: [...state.items, { ...action.item, quantity: 1 }],
      }
    }
    case 'REMOVE_ITEM':
      return {
        ...state,
        items: state.items.filter((i) => i.id !== action.id),
      }
    case 'INCREMENT':
      return {
        ...state,
        items: state.items.map((i) =>
          i.id === action.id ? { ...i, quantity: i.quantity + 1 } : i
        ),
      }
    case 'DECREMENT':
      return {
        ...state,
        items: state.items
          .map((i) => i.id === action.id ? { ...i, quantity: i.quantity - 1 } : i)
          .filter((i) => i.quantity > 0),
      }
    case 'CLEAR':
      return { ...state, items: [] }
    case 'TOGGLE_CART':
      return { ...state, isOpen: !state.isOpen }
  }
}

const PRODUCTS = [
  { id: 1, name: 'Teclado mecánico',  price: 89  },
  { id: 2, name: 'Monitor 27"',       price: 349 },
  { id: 3, name: 'Mouse inalámbrico', price: 29  },
  { id: 4, name: 'Webcam HD',         price: 59  },
]

export default function ShoppingCart() {
  const [cart, dispatch] = useReducer(cartReducer, { items: [], isOpen: false })

  const total     = useMemo(() => cart.items.reduce((acc, i) => acc + i.price * i.quantity, 0), [cart.items])
  const itemCount = useMemo(() => cart.items.reduce((acc, i) => acc + i.quantity, 0),           [cart.items])

  return (
    <div style={{ maxWidth: 440, fontFamily: 'sans-serif' }}>

      {/* Catálogo */}
      <div style={{ marginBottom: 16 }}>
        {PRODUCTS.map((product) => (
          <div
            key={product.id}
            style={{
              display: 'flex', justifyContent: 'space-between',
              alignItems: 'center', padding: '10px 0',
              borderBottom: '1px solid #e5e7eb',
            }}
          >
            <div>
              <p style={{ margin: 0, fontWeight: 500 }}>{product.name}</p>
              <p style={{ margin: 0, fontSize: 13, color: '#6b7280' }}>${product.price}</p>
            </div>
            <button
              onClick={() => dispatch({ type: 'ADD_ITEM', item: product })}
              style={{
                padding: '6px 14px', background: '#0070f3', color: '#fff',
                border: 'none', borderRadius: 6, cursor: 'pointer',
              }}
            >
              + Agregar
            </button>
          </div>
        ))}
      </div>

      {/* Botón carrito */}
      <button
        onClick={() => dispatch({ type: 'TOGGLE_CART' })}
        style={{
          width: '100%', padding: '10px',
          background: itemCount > 0 ? '#0070f3' : '#f3f4f6',
          color:      itemCount > 0 ? '#fff'    : '#6b7280',
          border: 'none', borderRadius: 8, cursor: 'pointer',
          fontWeight: 600, marginBottom: 12,
        }}
      >
        {cart.isOpen ? 'Ocultar carrito' : `Ver carrito (${itemCount} items)`}
      </button>

      {/* Panel del carrito */}
      {cart.isOpen && (
        <div style={{ border: '1px solid #e5e7eb', borderRadius: 10, padding: 16 }}>
          {cart.items.length === 0 ? (
            <p style={{ color: '#9ca3af', margin: 0 }}>El carrito está vacío.</p>
          ) : (
            <>
              {cart.items.map((item) => (
                <div
                  key={item.id}
                  style={{
                    display: 'flex', justifyContent: 'space-between',
                    alignItems: 'center', padding: '8px 0',
                    borderBottom: '1px solid #f3f4f6',
                  }}
                >
                  <span style={{ fontSize: 14, flex: 1 }}>{item.name}</span>
                  <div style={{ display: 'flex', alignItems: 'center', gap: 6 }}>
                    <button
                      onClick={() => dispatch({ type: 'DECREMENT', id: item.id })}
                      style={qtyBtn}
                    >
                      −
                    </button>
                    <span style={{ minWidth: 20, textAlign: 'center', fontSize: 14 }}>
                      {item.quantity}
                    </span>
                    <button
                      onClick={() => dispatch({ type: 'INCREMENT', id: item.id })}
                      style={qtyBtn}
                    >
                      +
                    </button>
                    <span style={{ minWidth: 60, textAlign: 'right', fontSize: 14 }}>
                      ${(item.price * item.quantity).toFixed(2)}
                    </span>
                    <button
                      onClick={() => dispatch({ type: 'REMOVE_ITEM', id: item.id })}
                      style={{ background: 'none', border: 'none', cursor: 'pointer', color: '#ef4444' }}
                    >
                      ✕
                    </button>
                  </div>
                </div>
              ))}

              <div style={{ paddingTop: 12, display: 'flex', justifyContent: 'space-between' }}>
                <span style={{ fontWeight: 600 }}>Total</span>
                <span style={{ fontWeight: 700, fontSize: 16 }}>${total.toFixed(2)}</span>
              </div>

              <button
                onClick={() => dispatch({ type: 'CLEAR' })}
                style={{
                  marginTop: 12, width: '100%', padding: '8px',
                  background: '#fee2e2', color: '#991b1b',
                  border: 'none', borderRadius: 6, cursor: 'pointer',
                }}
              >
                Vaciar carrito
              </button>
            </>
          )}
        </div>
      )}
    </div>
  )
}

const qtyBtn: React.CSSProperties = {
  width: 24, height: 24, border: '1px solid #d1d5db',
  borderRadius: 4, background: '#f9fafb',
  cursor: 'pointer', fontSize: 14, lineHeight: 1,
}
```

### Prueba esto

- Haz clic en "+ Agregar" en el mismo producto varias veces — la cantidad sube en lugar de añadir duplicados; el case `ADD_ITEM` detecta `exists` y actualiza `quantity` en lugar de insertar un nuevo item
- Abre el carrito y pulsa `−` hasta llegar a 0 en un producto — el item desaparece automáticamente porque `DECREMENT` encadena `.filter((i) => i.quantity > 0)` en la misma acción
- Añade varios productos y pulsa "Vaciar carrito" — todos desaparecen de una vez y el total vuelve a 0; observa que `CLEAR` retorna `{ ...state, items: [] }` y `isOpen` se mantiene
- Cambia el precio del Monitor de `349` a `0` en `PRODUCTS` — el total se recalcula automáticamente porque `total` usa `useMemo` derivado del estado del reducer
- Añade un quinto producto al array `PRODUCTS` sin tocar el reducer ni el JSX — aparece en el catálogo de inmediato; el componente es genérico respecto a los datos
- Observa el botón "Ver carrito" cuando el carrito está vacío vs. con items — el color cambia porque usa `itemCount > 0` como condición; prueba con exactamente 1 item para ver la transición
- Desde aquí puedes volver a cualquier PASO anterior cambiando el número y guardando.

### Agrega a `src/App.tsx`

```tsx
import ShoppingCart from './components/ShoppingCart'
```

```tsx
PASO === 3 ? <ShoppingCart /> :
```

Cambia `PASO = 3` y guarda.

---

## Cómo TypeScript estrecha tipos en el `switch`

```ts
type Action =
  | { type: 'ADD';    payload: string }
  | { type: 'REMOVE'; id: number }
  | { type: 'CLEAR' }

function reducer(state: string[], action: Action): string[] {
  switch (action.type) {
    case 'ADD':
      // TypeScript sabe: action = { type: 'ADD', payload: string }
      return [...state, action.payload]  // ✅ .payload disponible

    case 'REMOVE':
      // TypeScript sabe: action = { type: 'REMOVE', id: number }
      return state.filter((_, i) => i !== action.id)  // ✅ .id disponible

    case 'CLEAR':
      // TypeScript sabe: action = { type: 'CLEAR' }
      // action.payload → ❌ Error — no existe en este tipo
      return []
  }
}
```

Si olvidas un `case`, TypeScript lanza un error porque la función puede retornar
`undefined` — la cobertura exhaustiva del `switch` es obligatoria con `strict: true`.

---

## Errores comunes con `useReducer`

```ts
// ❌ Mutación directa del estado — React no detecta el cambio
case 'ADD_ITEM':
  state.items.push(action.item)  // muta el array original
  return state                   // misma referencia — no hay re-render

// ✅ Siempre retorna un objeto nuevo
case 'ADD_ITEM':
  return { ...state, items: [...state.items, action.item] }

// ❌ Olvidar spread del estado — pierdes otros campos
case 'SET_NAME':
  return { name: action.value }  // perdiste email, password, errors, status

// ✅ Spread primero, luego sobrescribe
case 'SET_NAME':
  return { ...state, name: action.value }
```

---

## Ejercicios propuestos

1. **`TrafficLight`** — semáforo con tres estados (`red`, `yellow`, `green`).
   Acciones: `NEXT` (avanza al siguiente color en el ciclo) y `RESET`.
   Usa un `useReducer` con unión discriminada de un solo campo `color`.

2. **`TextEditor`** — editor de texto con historial de deshacer.
   Estado: `{ text: string, history: string[] }`.
   Acciones: `SET_TEXT`, `UNDO`, `CLEAR`.
   `UNDO` restaura el último texto del historial.

3. **`MultiStepForm`** — formulario en 3 pasos con navegación.
   Estado: `{ step: 1 | 2 | 3, data: { ... } }`.
   Acciones: `NEXT_STEP`, `PREV_STEP`, `SET_FIELD`, `SUBMIT`.

---

## Resumen de la página 7

- `useReducer` centraliza la lógica de estado en una función pura — ideal cuando varios campos cambian juntos o hay múltiples transiciones relacionadas.
- El tipo de las acciones es una **unión discriminada** de objetos con un campo `type` literal — TypeScript estrecha el tipo automáticamente dentro de cada `case`.
- El reducer **nunca muta el estado** — siempre retorna un objeto nuevo con spread.
- Si un `case` no retorna, TypeScript lanza un error con `strict: true` — la cobertura del `switch` es exhaustiva por diseño.
- `Omit<CartItem, 'quantity'>` permite reutilizar interfaces existentes sin duplicar tipos.
- `useReducer` y `useMemo` se combinan naturalmente: el reducer gestiona las transiciones, `useMemo` deriva valores calculados del estado.

---

> **Siguiente página →** `useContext`: compartir estado entre componentes
> sin prop drilling, tipado con TypeScript y patrones de Context + Reducer.