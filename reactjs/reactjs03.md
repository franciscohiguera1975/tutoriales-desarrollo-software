# Curso React.js + TypeScript — Página 3
## Módulo 1 · Fundamentos
### Funciones como props, eventos tipados, `children` y composición padre ↔ hijo

---

## Funciones como props: el concepto

En React el flujo de datos es **unidireccional**: los datos bajan del padre al hijo a través de props. Pero ¿cómo le comunica el hijo algo al padre? A través de **funciones que el padre define y pasa como prop al hijo**.

El hijo nunca modifica el estado del padre directamente. En su lugar, llama a la función que recibió como prop, y es el padre quien decide qué hacer con esa llamada.

```
Padre
 │  define la función y la pasa como prop ↓
 ▼
Hijo
 │  llama a la función cuando ocurre algo ↑
 ▼
Padre reacciona (actualiza estado, hace fetch, navega, etc.)
```

Este patrón es fundamental. Todo lo que en otros frameworks sería un "evento personalizado" o un "emit", en React es simplemente **una función pasada como prop**.

---

## Tipar funciones como props

```ts
// Sin parámetros, sin retorno — el caso más frecuente
onClose: () => void

// Con un parámetro
onSearch: (query: string) => void

// Con múltiples parámetros
onSelect: (id: number, label: string) => void

// Función que retorna un valor
onValidate: (value: string) => boolean

// Alias reutilizable
type IdCallback = (id: number) => void

interface ProductListProps {
  onDelete: IdCallback
  onEdit:   IdCallback
}
```

> `() => void` es el tipo correcto para callbacks de UI que no retornan un valor significativo.

---

## Eventos del DOM tipados

Cuando usas tipos `React.*` en un archivo, **debes importar React**:

```tsx
import React from 'react'

function SearchInput() {
  function handleChange(e: React.ChangeEvent<HTMLInputElement>): void {
    console.log(e.target.value)
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>): void {
    e.preventDefault()
  }

  function handleKeyDown(e: React.KeyboardEvent<HTMLInputElement>): void {
    if (e.key === 'Enter') { /* ... */ }
  }

  return <input onChange={handleChange} onKeyDown={handleKeyDown} />
}
```

El tipo del elemento entre `< >` determina qué propiedades están disponibles.
`e.target.value` solo existe en `HTMLInputElement`, no en `HTMLDivElement`.

---

## Error clásico: pasar vs invocar

```tsx
// ❌ Error — ejecuta la función al renderizar, no al hacer clic
<ActionButton onAction={handleSave()} />

// ✅ Correcto — pasa la referencia
<ActionButton onAction={handleSave} />

// ✅ Con argumentos — función anónima como envoltura
<ActionButton onAction={() => handleDelete(product.id)} />
```

---

## La prop especial `children`

`children` es la prop que React reserva para el contenido entre las etiquetas de apertura y cierre.
Se tipa con `React.ReactNode`, que requiere importar React:

```tsx
import React from 'react'

interface ContentCardProps {
  title: string
  children: React.ReactNode
}

export default function ContentCard({ title, children }: ContentCardProps) {
  return (
    <div>
      <h3>{title}</h3>
      {children}
    </div>
  )
}
```

```tsx
// Uso — lo que va entre las etiquetas se convierte en children
<ContentCard title="Mi sección">
  <p>Este párrafo es children</p>
  <ActionButton label="Acción" onAction={() => {}} />
</ContentCard>
```

---

## Archivos completos del ejemplo

El ejemplo construye cinco componentes entrelazados:
`UserAvatar` → `UserSessionHeader` → `UserDashboardPanel` → `App`
más `ActionButton` y `ContentCard` como piezas reutilizables.

---

### `src/components/UserAvatar.tsx`

```tsx
// src/components/UserAvatar.tsx
// No usa tipos React.* → no necesita importar React

interface UserAvatarProps {
  fullName: string
  src?: string
  size?: number
}

export default function UserAvatar({
  fullName,
  src,
  size = 40,
}: UserAvatarProps) {
  const initials = fullName
    .split(' ')
    .map((word) => word[0])
    .join('')
    .toUpperCase()
    .slice(0, 2)

  if (src) {
    return (
      <img
        src={src}
        alt={`Avatar de ${fullName}`}
        style={{
          width: size,
          height: size,
          borderRadius: '50%',
          objectFit: 'cover',
        }}
      />
    )
  }

  return (
    <div
      style={{
        width: size,
        height: size,
        borderRadius: '50%',
        backgroundColor: '#0070f3',
        color: '#fff',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        fontWeight: 600,
        fontSize: size * 0.35,
        flexShrink: 0,
      }}
    >
      {initials}
    </div>
  )
}
```

### Prueba esto

- Pasa `size={80}` — el avatar crece; el `fontSize` interno se calcula como `size * 0.35`, manteniéndose proporcional
- Pasa `fullName="María López Ruiz"` — las iniciales calculadas son `"ML"` por el `.slice(0, 2)` al final
- Añade `src="https://i.pravatar.cc/80"` — el componente muestra `<img>` en lugar del círculo de iniciales gracias al `if (src)`
- Cambia `backgroundColor: '#0070f3'` a `'#7c3aed'` — todos los avatares sin `src` reflejan el nuevo color
- Añade `ring?: boolean` a las props y, cuando sea `true`, añade `outline: '3px solid #0070f3'` al estilo del `<div>`
- Pasa `fullName="A"` — el avatar muestra solo `"A"`; el cálculo de iniciales funciona con cualquier longitud de nombre

---

### `src/components/ActionButton.tsx`

```tsx
// src/components/ActionButton.tsx
// Usa React.CSSProperties → importa React

import React from 'react'

type ButtonVariant = 'primary' | 'secondary' | 'danger'

interface ActionButtonProps {
  label: string
  onAction: () => void
  variant?: ButtonVariant
  disabled?: boolean
}

export default function ActionButton({
  label,
  onAction,
  variant = 'primary',
  disabled = false,
}: ActionButtonProps) {

  const baseStyle: React.CSSProperties = {
    padding: '8px 16px',
    borderRadius: 6,
    border: 'none',
    cursor: disabled ? 'not-allowed' : 'pointer',
    fontWeight: 500,
    fontSize: 14,
    opacity: disabled ? 0.5 : 1,
  }

  const variantStyles: Record<ButtonVariant, React.CSSProperties> = {
    primary:   { backgroundColor: '#0070f3', color: '#fff' },
    secondary: { backgroundColor: '#eee',    color: '#333' },
    danger:    { backgroundColor: '#e00',    color: '#fff' },
  }

  return (
    <button
      onClick={onAction}
      disabled={disabled}
      style={{ ...baseStyle, ...variantStyles[variant] }}
    >
      {label}
    </button>
  )
}
```

### Prueba esto

- Cambia `variant="danger"` a `"secondary"` en `App.tsx` — el botón cambia de rojo a gris sin tocar el componente
- Pasa `disabled={true}` — el cursor cambia a `not-allowed` y la opacidad baja a `0.5` gracias al operador ternario en `baseStyle`
- Añade `'ghost'` al tipo `ButtonVariant` y su entrada en `variantStyles`: `{ backgroundColor: 'transparent', color: '#0070f3' }` — TypeScript marcará error hasta que el `Record` esté completo
- Cambia `padding: '8px 16px'` a `'12px 24px'` — el botón crece; experimenta con otros valores de padding
- Añade `icon?: string` a las props y renderiza `{icon && <span style={{ marginRight: 6 }}>{icon}</span>}` antes del `{label}`
- Pasa `onAction={handleSave()}` (con paréntesis) — observa cómo el botón ejecuta la función al renderizar, no al hacer clic; es el error más frecuente con callbacks

---

### `src/components/ContentCard.tsx`

```tsx
// src/components/ContentCard.tsx
// Usa React.ReactNode → importa React

import React from 'react'

interface ContentCardProps {
  title: string
  children: React.ReactNode
}

export default function ContentCard({
  title,
  children,
}: ContentCardProps) {
  return (
    <div
      style={{
        border: '1px solid #ddd',
        borderRadius: 10,
        padding: 20,
        marginBottom: 16,
      }}
    >
      <h3 style={{ margin: '0 0 12px', fontSize: 16 }}>{title}</h3>
      <div>{children}</div>
    </div>
  )
}
```

### Prueba esto

- Pasa diferentes elementos como `children`: un `<ul>`, un `<img>`, texto plano — `React.ReactNode` acepta cualquier JSX válido
- Anida dos `<ContentCard>` una dentro de la otra — `children` puede incluir otros componentes, incluyendo el mismo componente
- Añade `subtitle?: string` a las props y muéstralo entre el `<h3>` y el `<div>{children}</div>` con color gris
- Cambia `padding: 20` a `40` y `borderRadius: 10` a `0` — la tarjeta se vuelve más compacta y sin esquinas redondeadas
- Usa `children` como función: `children?: (title: string) => React.ReactNode` — esto es el patrón "render prop"; pásalo como `{(t) => <p>{t}</p>}`
- Añade `footer?: React.ReactNode` como segunda área de contenido debajo del `<div>{children}</div>`

---

### `src/components/UserSessionHeader.tsx`

```tsx
// src/components/UserSessionHeader.tsx
// No usa tipos React.* directamente → no necesita importar React
// Sí importa sus componentes hijos

import UserAvatar   from './UserAvatar'
import ActionButton from './ActionButton'

interface UserSessionHeaderProps {
  fullName: string
  role: string
  avatarSrc?: string
  onLogout: () => void
}

export default function UserSessionHeader({
  fullName,
  role,
  avatarSrc,
  onLogout,
}: UserSessionHeaderProps) {
  return (
    <div
      style={{
        display: 'flex',
        alignItems: 'center',
        gap: 12,
        padding: '12px 0',
        borderBottom: '1px solid #eee',
        marginBottom: 20,
      }}
    >
      <UserAvatar fullName={fullName} src={avatarSrc} />

      <div style={{ flexGrow: 1 }}>
        <p style={{ margin: 0, fontWeight: 600 }}>{fullName}</p>
        <p style={{ margin: 0, fontSize: 12, color: '#888' }}>{role}</p>
      </div>

      <ActionButton
        label="Cerrar sesión"
        variant="secondary"
        onAction={onLogout}
      />
    </div>
  )
}
```

### Prueba esto

- Cambia `variant="secondary"` a `"danger"` en el `ActionButton` de logout — el botón se vuelve rojo sin tocar el componente padre
- Pasa `avatarSrc="https://i.pravatar.cc/80"` desde `App.tsx` — `UserSessionHeader` reenvía la prop a `UserAvatar` que muestra la imagen
- Añade `lastSeen?: string` a las props y muéstralo debajo del rol: `<p style={{ margin: 0, fontSize: 11, color: '#bbb' }}>{lastSeen}</p>`
- Cambia `gap: 12` a `32` en el div externo — más espacio entre avatar, nombre y botón de logout
- Observa que este componente no importa React aunque renderiza JSX — porque no usa ningún tipo `React.*`; sus hijos se importan directamente

---

### `src/components/UserDashboardPanel.tsx`

```tsx
// src/components/UserDashboardPanel.tsx
// No usa tipos React.* directamente → no necesita importar React

import UserSessionHeader from './UserSessionHeader'
import ContentCard       from './ContentCard'
import ActionButton      from './ActionButton'

interface CurrentUser {
  fullName: string
  role: string
  avatarSrc?: string
}

interface UserDashboardPanelProps {
  currentUser: CurrentUser
  onLogout: () => void
  onCreateProject: () => void
}

export default function UserDashboardPanel({
  currentUser,
  onLogout,
  onCreateProject,
}: UserDashboardPanelProps) {
  return (
    <div style={{ maxWidth: 560, margin: '0 auto' }}>

      <UserSessionHeader
        fullName={currentUser.fullName}
        role={currentUser.role}
        avatarSrc={currentUser.avatarSrc}
        onLogout={onLogout}
      />

      <ContentCard title="Panel de control">
        <p style={{ marginTop: 0 }}>
          Bienvenido de vuelta, <strong>{currentUser.fullName}</strong>.
        </p>
        <ActionButton
          label="+ Nuevo proyecto"
          onAction={onCreateProject}
        />
      </ContentCard>

      <ContentCard title="Actividad reciente">
        <p style={{ color: '#888', fontSize: 14 }}>No hay actividad reciente.</p>
      </ContentCard>

    </div>
  )
}
```

### Prueba esto

- Haz clic en el botón "Nuevo proyecto" en la app — llama a `handleCreateProject` en `App.tsx`; observa cómo el callback sube del nieto al abuelo
- Añade un tercer `<ContentCard title="Tareas pendientes">` con una lista `<ul>` hardcodeada de tres tareas
- Cambia el saludo `currentUser.fullName` a `currentUser.fullName.split(' ')[0]` — muestra solo el primer nombre
- Añade `isAdmin?: boolean` a `CurrentUser` y condiciona un badge `"Admin"` junto al nombre en el header
- Observa el árbol de imports: `App → UserDashboardPanel → { UserSessionHeader → { UserAvatar, ActionButton }, ContentCard, ActionButton }` — cada componente es responsable de importar sus hijos

---

### `src/App.tsx` — navegador de pasos

```tsx
// src/App.tsx

import UserAvatar          from './components/UserAvatar'
import ActionButton        from './components/ActionButton'
import ContentCard         from './components/ContentCard'
import UserSessionHeader   from './components/UserSessionHeader'
import UserDashboardPanel  from './components/UserDashboardPanel'
import CatalogProductItem  from './components/CatalogProductItem'
import ShoppingCartSummary from './components/ShoppingCartSummary'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  UserAvatar           — avatar con iniciales o imagen            │
// │  2  ActionButton         — botón tipado con variantes               │
// │  3  ContentCard          — composición con children                 │
// │  4  UserSessionHeader    — UserAvatar + ActionButton                │
// │  5  UserDashboardPanel   — panel completo integrado                 │
// │  6  CatalogProductItem   — item con callback onAddToCart            │
// │  7  ShoppingCartSummary  — resumen con callback onClearCart         │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

const catalog = [
  { id: 1, name: 'Teclado mecánico',  price: 89.99 },
  { id: 2, name: 'Monitor 27"',       price: 349.99 },
  { id: 3, name: 'Mouse inalámbrico', price: 29.99 },
]

const cartItems = [
  { id: 2, name: 'Monitor 27"', price: 349.99 },
]

export default function App() {
  function handleLogout(): void {
    alert('Sesión cerrada. Redirigiendo al login...')
  }

  function handleCreateProject(): void {
    alert('Abriendo formulario de nuevo proyecto...')
  }

  const content =
    PASO === 1 ? <UserAvatar fullName="Ana García" size={64} /> :
    PASO === 2 ? (
      <div style={{ display: 'flex', gap: 8 }}>
        <ActionButton label="Primario"   onAction={() => alert('primario')} />
        <ActionButton label="Secundario" onAction={() => {}} variant="secondary" />
        <ActionButton label="Peligro"    onAction={() => {}} variant="danger" />
        <ActionButton label="Desactivado" onAction={() => {}} disabled />
      </div>
    ) :
    PASO === 3 ? (
      <ContentCard title="Sección de ejemplo">
        <p>Este párrafo es el children de ContentCard.</p>
        <ActionButton label="Acción" onAction={() => alert('click')} />
      </ContentCard>
    ) :
    PASO === 4 ? (
      <UserSessionHeader
        fullName="Ana García"
        role="Administradora"
        onLogout={handleLogout}
      />
    ) :
    PASO === 5 ? (
      <UserDashboardPanel
        currentUser={{ fullName: 'Ana García', role: 'Administradora' }}
        onLogout={handleLogout}
        onCreateProject={handleCreateProject}
      />
    ) :
    PASO === 6 ? (
      <section>
        {catalog.map((p) => (
          <CatalogProductItem
            key={p.id}
            id={p.id}
            name={p.name}
            price={p.price}
            onAddToCart={(id, name, price) => alert(`Agregado: ${name} — $${price}`)}
          />
        ))}
      </section>
    ) :
    PASO === 7 ? (
      <ShoppingCartSummary
        items={cartItems}
        onClearCart={() => alert('Carrito vaciado')}
      />
    ) :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 560, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

---

### Flujo de funciones e imports en el árbol

```
App
 ├─ imports: UserDashboardPanel
 ├─ define: handleLogout, handleCreateProject
 └─ UserDashboardPanel
      ├─ imports: UserSessionHeader, ContentCard, ActionButton
      ├─ UserSessionHeader
      │    ├─ imports: UserAvatar, ActionButton
      │    └─ UserAvatar      (sin imports de componentes)
      └─ ContentCard
           ├─ imports: React  (por React.ReactNode)
           └─ ActionButton
                └─ imports: React  (por React.CSSProperties)
```

---

## Ejercicio de la página 3

**Objetivo**: practicar funciones como props, callbacks con datos, eventos tipados y composición.

### Enunciado

Construye un sistema de carrito de compras mínimo con tres componentes.

**`CatalogProductItem`** — muestra un producto con botón "Agregar":
- Props: `id: number`, `name: string`, `price: number`, `onAddToCart: (id: number, name: string, price: number) => void`
- Al hacer clic llama a `onAddToCart` con los datos del producto.

**`ShoppingCartSummary`** — muestra los items del carrito:
- Props: `items: Array<{ id: number, name: string, price: number }>`, `onClearCart: () => void`
- Muestra cada item, el total calculado y un botón "Vaciar carrito".

**`App`** — orquesta todo:
- Mantiene por ahora un array hardcodeado de items (en la página 4 será dinámico con `useState`).
- Define `handleAddToCart` y `handleClearCart` con `console.log` por ahora.

### Solución de referencia

```tsx
// src/components/CatalogProductItem.tsx
// No usa tipos React.* → no necesita importar React

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

```tsx
// src/components/ShoppingCartSummary.tsx
// No usa tipos React.* → no necesita importar React

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
            style={{
              display: 'flex',
              justifyContent: 'space-between',
              padding: '6px 0',
            }}
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

```tsx
// src/App.tsx
// No usa tipos React.* → no necesita importar React

import CatalogProductItem  from './components/CatalogProductItem'
import ShoppingCartSummary from './components/ShoppingCartSummary'

interface CartItem {
  id: number
  name: string
  price: number
}

// Array estático por ahora — se vuelve dinámico con useState en la página 4
const cartItems: CartItem[] = [
  { id: 2, name: 'Monitor 27"', price: 349.99 },
]

const catalog = [
  { id: 1, name: 'Teclado mecánico',  price: 89.99 },
  { id: 2, name: 'Monitor 27"',       price: 349.99 },
  { id: 3, name: 'Mouse inalámbrico', price: 29.99 },
]

export default function App() {

  function handleAddToCart(id: number, name: string, price: number): void {
    console.log(`Agregando: [${id}] ${name} — $${price}`)
    // Página 4: setCartItems(prev => [...prev, { id, name, price }])
  }

  function handleClearCart(): void {
    console.log('Vaciando carrito...')
    // Página 4: setCartItems([])
  }

  return (
    <main style={{ maxWidth: 480, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      <h1 style={{ fontSize: 22 }}>Tienda</h1>

      <section>
        {catalog.map((product) => (
          <CatalogProductItem
            key={product.id}
            id={product.id}
            name={product.name}
            price={product.price}
            onAddToCart={handleAddToCart}
          />
        ))}
      </section>

      <ShoppingCartSummary
        items={cartItems}
        onClearCart={handleClearCart}
      />
    </main>
  )
}
```

> **Nota**: el carrito aún no es interactivo. En la página 4 retomamos este ejercicio
> y lo hacemos completamente funcional con `useState`.

### Prueba esto

- Abre la consola del navegador (F12) y haz clic en "Agregar" en varios productos — verás los `console.log` del `handleAddToCart` con el id, nombre y precio
- Cambia `cartItems` a `[]` — el `ShoppingCartSummary` muestra el mensaje "El carrito está vacío." y oculta el total y el botón
- Añade `{ id: 3, name: 'Mouse inalámbrico', price: 29.99 }` a `cartItems` — el total se recalcula automáticamente con `.reduce()`
- Añade `quantity?: number` a `CartItem` en `ShoppingCartSummary` y muéstralo en cada fila: `{item.quantity ?? 1} × {item.name}`
- Cambia el color del botón "Agregar" en `CatalogProductItem` de `'#0070f3'` a `'#16a34a'` — el cambio aplica a todos los botones de la lista

---

## Resumen de la página 3

- Los hijos se comunican con el padre llamando a **funciones que el padre les pasó como props**.
- Las funciones se tipan con `(param: Tipo) => void` o el tipo de retorno que corresponda.
- **Importar React es necesario cuando usas tipos del namespace `React.*`** como `React.CSSProperties`, `React.ReactNode`, `React.ChangeEvent<T>`, etc. Para JSX puro no hace falta.
- Cada componente importa explícitamente los componentes hijos que renderiza.
- `children: React.ReactNode` permite envolver contenido arbitrario — requiere `import React`.
- Las funciones se definen arriba en el árbol y viajan hacia abajo como props.
- `onClick={fn()}` ejecuta la función al renderizar — error más frecuente con callbacks.

---

> **Siguiente página →** `useState` en profundidad: tipos inferidos vs explícitos,
> estado con objetos y arrays, y el carrito de esta página hecho completamente funcional.