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

> Establece `const PASO = 1` en `App.tsx` y guarda — verás `<UserAvatar fullName="Ana García" size={64} />` renderizado en pantalla.

- En el `<UserAvatar>` de `App.tsx` (PASO 1), cambia `size={64}` a `size={80}` — el avatar crece; el `fontSize` interno se calcula como `size * 0.35`, manteniéndose proporcional
- En el `<UserAvatar>` de `App.tsx`, cambia `fullName="Ana García"` a `fullName="María López Ruiz"` — las iniciales calculadas son `"ML"` por el `.slice(0, 2)` al final
- En el `<UserAvatar>` de `App.tsx`, añade `src="https://i.pravatar.cc/80"` — el componente muestra `<img>` en lugar del círculo de iniciales gracias al `if (src)`
- En `UserAvatar.tsx`, cambia `backgroundColor: '#0070f3'` a `'#7c3aed'` — todos los avatares sin `src` reflejan el nuevo color
- En `UserAvatar.tsx`, añade `ring?: boolean` a las props y, cuando sea `true`, añade `outline: '3px solid #0070f3'` al estilo del `<div>`
- En el `<UserAvatar>` de `App.tsx`, cambia `fullName="Ana García"` a `fullName="A"` — el avatar muestra solo `"A"`; el cálculo de iniciales funciona con cualquier longitud de nombre

### `src/App.tsx`

```tsx
// src/App.tsx

import UserAvatar from './components/UserAvatar'
// ┌──────────────────────────────────────────────────────────────────────┐
// │  1  UserAvatar          — avatar con iniciales o imagen             │
// │  2  ActionButton        — botón tipado con variantes                │
// │  3  ContentCard         — composición con children                  │
// │  4  UserSessionHeader   — UserAvatar + ActionButton                 │
// │  5  UserDashboardPanel  — panel completo integrado                  │
// │  6  RatingStars         — 5 estrellas con callback onRate           │
// │  7  ConfirmDialog       — diálogo con callbacks onConfirm/onCancel  │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content = PASO === 1 ? <UserAvatar fullName="Ana García" size={64} /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 560, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

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

> Establece `const PASO = 2` en `App.tsx` y guarda — verás cuatro botones (primario, secundario, peligro y desactivado) en pantalla.

- En `App.tsx` (PASO 2), cambia `variant="danger"` a `"secondary"` en el tercer `<ActionButton>` — el botón cambia de rojo a gris sin tocar el componente
- En `App.tsx`, pasa `disabled={true}` al primer `<ActionButton>` — el cursor cambia a `not-allowed` y la opacidad baja a `0.5` gracias al operador ternario en `baseStyle`
- En `ActionButton.tsx`, añade `'ghost'` al tipo `ButtonVariant` y su entrada en `variantStyles`: `{ backgroundColor: 'transparent', color: '#0070f3' }` — TypeScript marcará error hasta que el `Record` esté completo
- En `ActionButton.tsx`, cambia `padding: '8px 16px'` a `'12px 24px'` — el botón crece; experimenta con otros valores de padding
- En `ActionButton.tsx`, añade `icon?: string` a las props y renderiza `{icon && <span style={{ marginRight: 6 }}>{icon}</span>}` antes del `{label}`
- En `App.tsx`, cambia `onAction={() => alert('primario')}` a `onAction={alert('primario')}` (sin la función flecha) — el alert se dispara al cargar la página, no al hacer clic; es el error más frecuente con callbacks

### Agrega a `src/App.tsx`

```tsx
import ActionButton from './components/ActionButton'
```

```tsx
PASO === 2 ? (
  <div style={{ display: 'flex', gap: 8 }}>
    <ActionButton label="Primario"    onAction={() => alert('primario')} />
    <ActionButton label="Secundario"  onAction={() => {}} variant="secondary" />
    <ActionButton label="Peligro"     onAction={() => {}} variant="danger" />
    <ActionButton label="Desactivado" onAction={() => {}} disabled />
  </div>
) :
```

Cambia `PASO = 2` y guarda.

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

> Establece `const PASO = 3` en `App.tsx` y guarda — verás `ContentCard` con un párrafo y un botón como `children`.

- En el `<ContentCard>` de `App.tsx` (PASO 3), reemplaza el `<p>` por un `<ul>` con tres ítems o un `<img>` — `React.ReactNode` acepta cualquier JSX válido
- En `App.tsx`, anida un segundo `<ContentCard>` dentro del primero como `children` — `children` puede incluir otros componentes, incluyendo el mismo componente
- En `ContentCard.tsx`, añade `subtitle?: string` a las props y muéstralo entre el `<h3>` y el `<div>{children}</div>` con color gris; en `App.tsx` pasa `subtitle="Subtítulo de prueba"`
- En `ContentCard.tsx`, cambia `padding: 20` a `40` y `borderRadius: 10` a `0` — la tarjeta se amplía y pierde las esquinas redondeadas
- En `ContentCard.tsx`, usa `children` como función: `children?: (title: string) => React.ReactNode` — es el patrón "render prop"; en `App.tsx` pásalo como `{(t) => <p>{t}</p>}`
- En `ContentCard.tsx`, añade `footer?: React.ReactNode` y renderízalo debajo del `<div>{children}</div>`; en `App.tsx` pasa `footer={<p>Pie de tarjeta</p>}`

### Agrega a `src/App.tsx`

```tsx
import ContentCard from './components/ContentCard'
```

```tsx
PASO === 3 ? (
  <ContentCard title="Sección de ejemplo">
    <p>Este párrafo es el children de ContentCard.</p>
    <ActionButton label="Acción" onAction={() => alert('click')} />
  </ContentCard>
) :
```

Cambia `PASO = 3` y guarda.

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

> Establece `const PASO = 4` en `App.tsx` y guarda — verás el encabezado con avatar de iniciales, nombre, rol y botón "Cerrar sesión".

- En `UserSessionHeader.tsx`, cambia `variant="secondary"` a `"danger"` en el `<ActionButton>` de logout — el botón se vuelve rojo sin tocar el componente padre
- En el `<UserSessionHeader>` de `App.tsx` (PASO 4), añade `avatarSrc="https://i.pravatar.cc/80"` — el componente reenvía la prop a `UserAvatar`, que muestra la imagen en lugar del círculo de iniciales
- En `UserSessionHeader.tsx`, añade `lastSeen?: string` a las props y muéstralo debajo del rol: `<p style={{ margin: 0, fontSize: 11, color: '#bbb' }}>{lastSeen}</p>`; en `App.tsx` pasa `lastSeen="Hace 2 horas"`
- En `UserSessionHeader.tsx`, cambia `gap: 12` a `32` en el div externo — más espacio entre avatar, nombre y botón de logout
- Observa que `UserSessionHeader.tsx` no importa React aunque renderiza JSX — porque no usa ningún tipo `React.*`; sus hijos se importan directamente

### Agrega a `src/App.tsx`

```tsx
import UserSessionHeader from './components/UserSessionHeader'
```

```tsx
PASO === 4 ? (
  <UserSessionHeader
    fullName="Ana García"
    role="Administradora"
    onLogout={() => alert('Sesión cerrada. Redirigiendo al login...')}
  />
) :
```

Cambia `PASO = 4` y guarda.

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

> Establece `const PASO = 5` en `App.tsx` y guarda — verás el panel completo con header, tarjeta de bienvenida y tarjeta de actividad reciente.

- En el navegador, haz clic en "Nuevo proyecto" — llama a `handleCreateProject` definido en `App.tsx`; observa cómo el callback sube del nieto al abuelo sin que los intermedios lo procesen
- En `UserDashboardPanel.tsx`, añade un tercer `<ContentCard title="Tareas pendientes">` con una lista `<ul>` hardcodeada de tres tareas
- En `UserDashboardPanel.tsx`, cambia `currentUser.fullName` a `currentUser.fullName.split(' ')[0]` en el saludo de la tarjeta — muestra solo el primer nombre
- En `UserDashboardPanel.tsx`, añade `isAdmin?: boolean` a la interfaz `CurrentUser` y condiciona un badge `"Admin"` en el header; en `App.tsx` pasa `currentUser={{ fullName: 'Ana García', role: 'Administradora', isAdmin: true }}`
- Observa el árbol de imports: `App → UserDashboardPanel → { UserSessionHeader → { UserAvatar, ActionButton }, ContentCard, ActionButton }` — cada componente importa explícitamente sus hijos

### Agrega a `src/App.tsx`

```tsx
import UserDashboardPanel from './components/UserDashboardPanel'
```

```tsx
PASO === 5 ? (
  <UserDashboardPanel
    currentUser={{ fullName: 'Ana García', role: 'Administradora' }}
    onLogout={() => alert('Sesión cerrada. Redirigiendo al login...')}
    onCreateProject={() => alert('Abriendo formulario de nuevo proyecto...')}
  />
) :
```

Cambia `PASO = 5` y guarda.

---

### `src/components/RatingStars.tsx`

```tsx
// src/components/RatingStars.tsx
// No usa tipos React.* → no necesita importar React

interface RatingStarsProps {
  rating: number
  onRate: (value: number) => void
  label?: string
}

export default function RatingStars({
  rating,
  onRate,
  label,
}: RatingStarsProps) {
  return (
    <div style={{ marginBottom: 12 }}>
      {label && (
        <p style={{ margin: '0 0 6px', fontSize: 14, color: '#555' }}>{label}</p>
      )}
      <div style={{ display: 'flex', gap: 4 }}>
        {[1, 2, 3, 4, 5].map((star) => (
          <button
            key={star}
            aria-label={`Dar ${star} estrella${star > 1 ? 's' : ''}`}
            onClick={() => onRate(star)}
            style={{
              fontSize: 28,
              background: 'none',
              border: 'none',
              cursor: 'pointer',
              color: star <= rating ? '#f59e0b' : '#d1d5db',
              padding: 0,
            }}
          >
            ★
          </button>
        ))}
      </div>
      <p style={{ margin: '6px 0 0', fontSize: 13, color: '#888' }}>
        {rating > 0 ? `${rating} / 5 estrellas` : 'Sin valorar'}
      </p>
    </div>
  )
}
```

### Prueba esto

> Establece `const PASO = 6` en `App.tsx` y guarda — verás las 5 estrellas con la primera resaltada; haz clic en cualquier estrella y revisa la consola.

- En `App.tsx` (PASO 6), cambia `rating={1}` a `rating={4}` — cuatro estrellas se muestran en amarillo; `star <= rating` decide el color en cada iteración del `.map()`
- En `App.tsx`, cambia `onRate={(n) => console.log('rating:', n)}` a `onRate={(n) => alert(\`Valoraste con ${n} estrellas\`)}` — la función anónima sigue siendo el padre quien decide qué hacer con el valor
- En `RatingStars.tsx`, cambia `fontSize: 28` a `40` — las estrellas crecen sin cambiar nada en `App.tsx`
- En `RatingStars.tsx`, cambia `star <= rating` a `star === rating` — solo la estrella exacta se ilumina; ilustra cómo la lógica interna no afecta la interfaz del componente
- Añade `label="Valora este producto:"` al `<RatingStars>` de `App.tsx` — el párrafo de etiqueta aparece sobre las estrellas gracias a la condición `{label && ...}`
- Observa que `RatingStars` no tiene estado propio: es un componente **controlado** — el padre lleva el valor actual (`rating`) y el hijo notifica cambios a través de `onRate`

### Agrega a `src/App.tsx`

```tsx
import RatingStars from './components/RatingStars'
```

```tsx
PASO === 6 ? (
  <RatingStars
    rating={1}
    onRate={(n) => console.log('rating:', n)}
    label="Valora este producto:"
  />
) :
```

Cambia `PASO = 6` y guarda.

---

### `src/components/ConfirmDialog.tsx`

```tsx
// src/components/ConfirmDialog.tsx
// No usa tipos React.* → no necesita importar React

interface ConfirmDialogProps {
  message: string
  onConfirm: () => void
  onCancel: () => void
  confirmLabel?: string
  cancelLabel?: string
}

export default function ConfirmDialog({
  message,
  onConfirm,
  onCancel,
  confirmLabel = 'Confirmar',
  cancelLabel  = 'Cancelar',
}: ConfirmDialogProps) {
  return (
    <div
      style={{
        border: '1px solid #ddd',
        borderRadius: 10,
        padding: 20,
        maxWidth: 360,
      }}
    >
      <p style={{ margin: '0 0 16px', fontSize: 15 }}>{message}</p>
      <div style={{ display: 'flex', gap: 8, justifyContent: 'flex-end' }}>
        <button
          onClick={onCancel}
          style={{
            padding: '8px 16px',
            borderRadius: 6,
            border: '1px solid #ddd',
            backgroundColor: '#f5f5f5',
            cursor: 'pointer',
          }}
        >
          {cancelLabel}
        </button>
        <button
          onClick={onConfirm}
          style={{
            padding: '8px 16px',
            borderRadius: 6,
            border: 'none',
            backgroundColor: '#e00',
            color: '#fff',
            cursor: 'pointer',
          }}
        >
          {confirmLabel}
        </button>
      </div>
    </div>
  )
}
```

### Prueba esto

> Establece `const PASO = 7` en `App.tsx` y guarda — verás el diálogo de confirmación con dos botones; haz clic en cada uno y observa los mensajes en la consola.

- En `App.tsx` (PASO 7), cambia `onConfirm={() => console.log('confirmado')}` a `onConfirm={() => alert('¡Acción realizada!')}` — el hijo llama a la función, pero el padre decide la consecuencia
- En `App.tsx`, pasa `confirmLabel="Sí, eliminar"` y `cancelLabel="No, volver"` — los textos de los botones cambian sin tocar el componente
- En `ConfirmDialog.tsx`, cambia `backgroundColor: '#e00'` a `'#16a34a'` en el botón de confirmar — el botón se vuelve verde; útil cuando la acción es positiva (p. ej., "Guardar cambios")
- En `ConfirmDialog.tsx`, añade `title?: string` a las props y renderiza un `<h3 style={{ margin: '0 0 8px' }}>` antes del `<p>` cuando esté presente; en `App.tsx` pasa `title="Confirmar eliminación"`
- Observa que `ConfirmDialog` tiene **dos** funciones como props distintas: `onConfirm` y `onCancel`; cada una con su propio tipo `() => void` — el componente no sabe qué hace cada una, solo las llama
- Desde aquí puedes volver a cualquier PASO anterior cambiando el número y guardando.

### Agrega a `src/App.tsx`

```tsx
import ConfirmDialog from './components/ConfirmDialog'
```

```tsx
PASO === 7 ? (
  <ConfirmDialog
    message="¿Seguro que quieres eliminar este elemento?"
    onConfirm={() => console.log('confirmado')}
    onCancel={() => console.log('cancelado')}
  />
) :
```

Cambia `PASO = 7` y guarda.

---

### Flujo de funciones e imports en el árbol

```
App
 ├─ imports: UserDashboardPanel, RatingStars, ConfirmDialog
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

**Objetivo**: practicar funciones como props, callbacks tipados y composición de componentes.

### Enunciado

Construye un sistema de valoraciones usando los dos componentes de la página.

**`FeedbackCard`** — muestra un producto con su valoración actual:
- Props: `name: string`, `rating: number`, `onRate: (n: number) => void`, `onDelete: () => void`
- Renderiza el nombre del producto, un `<RatingStars>` y un botón "✕" que llama a `onDelete`.

**`App`** — orquesta todo:
- Define un array `products` con tres productos (id, name, rating).
- Renderiza un `<FeedbackCard>` por producto.
- Bajo los productos, renderiza un `<ConfirmDialog>` con el mensaje "¿Borrar historial de valoraciones?".
- Los callbacks solo hacen `console.log` por ahora.

### Solución de referencia

```tsx
// src/components/FeedbackCard.tsx
// No usa tipos React.* → no necesita importar React

import RatingStars from './RatingStars'

interface FeedbackCardProps {
  name: string
  rating: number
  onRate: (n: number) => void
  onDelete: () => void
}

export default function FeedbackCard({
  name,
  rating,
  onRate,
  onDelete,
}: FeedbackCardProps) {
  return (
    <div
      style={{
        border: '1px solid #eee',
        borderRadius: 8,
        padding: 16,
        marginBottom: 12,
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'flex-start',
        gap: 12,
      }}
    >
      <div style={{ flexGrow: 1 }}>
        <p style={{ margin: '0 0 8px', fontWeight: 500 }}>{name}</p>
        <RatingStars rating={rating} onRate={onRate} />
      </div>
      <button
        onClick={onDelete}
        style={{
          background: 'none',
          border: 'none',
          color: '#e00',
          cursor: 'pointer',
          fontSize: 18,
          padding: 4,
        }}
        aria-label={`Eliminar ${name}`}
      >
        ✕
      </button>
    </div>
  )
}
```

```tsx
// src/App.tsx
// No usa tipos React.* → no necesita importar React

import FeedbackCard  from './components/FeedbackCard'
import ConfirmDialog from './components/ConfirmDialog'

const products = [
  { id: 1, name: 'Teclado mecánico',  rating: 4 },
  { id: 2, name: 'Monitor 27"',       rating: 5 },
  { id: 3, name: 'Mouse inalámbrico', rating: 3 },
]

export default function App() {
  return (
    <main style={{ maxWidth: 480, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      <h1 style={{ fontSize: 22 }}>Valoraciones</h1>

      {products.map((p) => (
        <FeedbackCard
          key={p.id}
          name={p.name}
          rating={p.rating}
          onRate={(n) => console.log(`${p.name}: ${n} ★`)}
          onDelete={() => console.log(`Eliminar: ${p.name}`)}
        />
      ))}

      <ConfirmDialog
        message="¿Borrar historial de valoraciones?"
        confirmLabel="Sí, borrar"
        onConfirm={() => console.log('Historial borrado')}
        onCancel={() => console.log('Cancelado')}
      />
    </main>
  )
}
```

### Prueba esto

- Abre la consola (F12 → Console) y haz clic en distintas estrellas de cada producto — verás `nombre: N ★` en la consola; `onRate` llega desde `App` pasando por `FeedbackCard` hasta `RatingStars`
- Haz clic en "✕" en cualquier `FeedbackCard` — la consola muestra `Eliminar: nombre`; la acción real (quitar del array) la implementaremos con `useState` en la página 4
- Haz clic en "Sí, borrar" en el `ConfirmDialog` — el callback `onConfirm` es definido en `App` y llega al componente como prop
- En `App.tsx`, añade `confirmLabel="Sí, eliminar todo"` al `<ConfirmDialog>` — el texto del botón cambia sin tocar el componente
- Observa cómo `FeedbackCard` importa `RatingStars` pero **no sabe nada sobre el padre** — el padre pasa los callbacks, el hijo los llama; esa es la clave de la comunicación unidireccional

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
> estado con objetos y arrays, y un carrito de compras completamente funcional.