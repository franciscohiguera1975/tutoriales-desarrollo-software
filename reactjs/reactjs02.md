# Curso React.js + TypeScript — Página 2
## Módulo 1 · Fundamentos
### JSX: definición, reglas, componentes funcionales tipados

---

## ¿Qué es JSX?

**JSX** (JavaScript XML) es una extensión sintáctica de JavaScript que permite escribir estructuras similares a HTML directamente dentro de código JS/TS. No es un lenguaje separado ni una plantilla: es azúcar sintáctico que el compilador transforma en llamadas a funciones JavaScript puras antes de que el navegador vea una sola línea de código.

En términos concretos: JSX es una forma de describir **qué debe renderizarse en pantalla** usando una sintaxis familiar (similar a HTML), pero con toda la potencia de JavaScript disponible dentro de ella.

Lo que tú escribes:

```tsx
const elemento = <h1 className="titulo">Hola mundo</h1>
```

El compilador lo transforma en:

```ts
const elemento = React.createElement('h1', { className: 'titulo' }, 'Hola mundo')
```

Y `React.createElement` retorna un **objeto JavaScript plano** que describe el nodo — no un nodo del DOM real. React compara ese objeto con el DOM actual y aplica solo los cambios necesarios. A ese objeto se le llama **elemento React**, y al proceso de comparación se le llama **reconciliación**.

> JSX no es HTML. No es un template engine. No es un lenguaje de marcado.
> Es JavaScript con una sintaxis extendida que describe árboles de UI.

### ¿Por qué existe JSX?

Sin JSX, construir interfaces en React sería así:

```ts
// Sin JSX — completamente válido pero ilegible a escala
React.createElement(
  'div',
  { className: 'tarjeta' },
  React.createElement('h2', null, 'Título'),
  React.createElement('p', { style: { color: 'gray' } }, 'Descripción'),
  React.createElement('button', { onClick: handleClick }, 'Aceptar')
)
```

JSX hace exactamente lo mismo pero de forma legible:

```tsx
// Con JSX — idéntico resultado, legible a cualquier escala
<div className="tarjeta">
  <h2>Título</h2>
  <p style={{ color: 'gray' }}>Descripción</p>
  <button onClick={handleClick}>Aceptar</button>
</div>
```

### JSX es opcional, pero universal

Técnicamente puedes escribir React sin JSX. En la práctica, el **100 % de los proyectos React reales lo usan** porque la diferencia en legibilidad y mantenibilidad es abismal.

---

## ¿Cuándo importar React?

Con `"jsx": "react-jsx"` en `tsconfig.json` **no necesitas importar React para usar JSX**. El compilador lo inyecta automáticamente.

Sin embargo, **sí necesitas importarlo cuando usas tipos del namespace `React.*`**:

```tsx
// ✅ No necesitas importar React solo por usar JSX
export default function MyComponent() {
  return <div>Hola</div>
}

// ✅ Sí necesitas importar React cuando usas sus tipos
import React from 'react'

export default function MyComponent() {
  const style: React.CSSProperties = { color: 'red' }
  return <div style={style}>Hola</div>
}
```

Tipos de React que requieren el import:

| Tipo | Cuándo se usa |
|---|---|
| `React.CSSProperties` | Tipar objetos de estilos inline |
| `React.ReactNode` | Prop `children` u otro contenido JSX arbitrario |
| `React.ChangeEvent<T>` | Eventos de inputs controlados |
| `React.MouseEvent<T>` | Eventos de clic con acceso al elemento |
| `React.FormEvent<T>` | Eventos de submit de formularios |
| `React.KeyboardEvent<T>` | Eventos de teclado |

---

## JSX y TypeScript: el archivo `.tsx`

- Archivos con JSX → extensión **`.tsx`**
- Archivos sin JSX (lógica pura, tipos, utilidades) → extensión **`.ts`**

Si intentas escribir JSX en un archivo `.ts`, el compilador lanza un error inmediatamente.

---

## Reglas fundamentales de JSX

### 1. Un único elemento raíz por retorno

```tsx
// ❌ Inválido — dos elementos raíz
return (
  <h1>Título</h1>
  <p>Párrafo</p>
)

// ✅ Opción A: envolver en un div
return (
  <div>
    <h1>Título</h1>
    <p>Párrafo</p>
  </div>
)

// ✅ Opción B: Fragment — no genera ningún nodo extra en el DOM
return (
  <>
    <h1>Título</h1>
    <p>Párrafo</p>
  </>
)
```

### 2. Diferencias con HTML

| HTML | JSX | Motivo |
|---|---|---|
| `class="..."` | `className="..."` | `class` es palabra reservada en JS |
| `for="..."` | `htmlFor="..."` | `for` es palabra reservada en JS |
| `<br>` | `<br />` | Toda etiqueta debe cerrarse explícitamente |
| `<input>` | `<input />` | Igual: auto-cerrada |
| `style="color:red"` | `style={{ color: 'red' }}` | El estilo es un objeto JS, no un string |
| `onclick="fn()"` | `onClick={fn}` | Eventos en camelCase; valor es una referencia |
| `tabindex` | `tabIndex` | Atributos en camelCase en JSX |

### 3. Expresiones dentro de `{ }`

```tsx
const usuario = 'Ana'
const precio  = 49.9
const activo  = true

return (
  <div>
    {/* Llamada a método */}
    <p>{usuario.toUpperCase()}</p>

    {/* Operación matemática */}
    <p>Con IVA: ${(precio * 1.12).toFixed(2)}</p>

    {/* Ternario para renderizado condicional */}
    <p>{activo ? 'Usuario activo' : 'Usuario inactivo'}</p>

    {/* && para renderizado condicional corto */}
    {activo && <span>✓ Verificado</span>}
  </div>
)
```

> JSX **no admite sentencias** dentro de `{ }`. `if`, `for`, `while` no pueden ir ahí directamente.

### 4. Componentes en PascalCase

```tsx
<div>           // → etiqueta HTML nativa
<button>        // → etiqueta HTML nativa
<ProductCard /> // → componente React (debe ir en PascalCase)
```

---

## Convención de export en componentes

`export default` va directamente en la declaración de la función — no como línea suelta al final:

```tsx
// ❌ Export separado al final
function ProductCard() { ... }
export default ProductCard

// ✅ Export en la declaración — patrón del curso
export default function ProductCard() { ... }
```

---

## `interface` vs `type` para props

```ts
// interface — describe la forma de un objeto, puede extenderse
interface ProductCardProps {
  title: string
  description?: string   // ? = prop opcional
  highlighted?: boolean
}

// Extensión con herencia
interface FeaturedProductCardProps extends ProductCardProps {
  badgeText: string
}

// type — para uniones y literales
type ButtonVariant = 'primary' | 'secondary' | 'danger'
```

| Situación | Recomendación |
|---|---|
| Props de un componente | `interface` |
| Unión de tipos literales | `type` |
| Tipos calculados o condicionales | `type` |

---

## Archivos completos del ejemplo

---

### `src/components/ProductCard.tsx`

```tsx
// src/components/ProductCard.tsx

interface ProductCardProps {
  title: string
  description?: string
  highlighted?: boolean
}

export default function ProductCard({
  title,
  description = 'Sin descripción',
  highlighted = false,
}: ProductCardProps) {
  return (
    <div
      style={{
        border: highlighted ? '2px solid gold' : '1px solid #ccc',
        borderRadius: 8,
        padding: 16,
        marginBottom: 12,
        backgroundColor: highlighted ? '#fffbea' : '#fff',
      }}
    >
      <h3 style={{ margin: '0 0 8px' }}>{title}</h3>
      <p style={{ margin: 0, color: '#555' }}>{description}</p>
    </div>
  )
}
```

> Este componente no usa tipos `React.*` directamente, por lo que no necesita importar React.
> Los estilos inline como `{ border: '...' }` son inferidos por TypeScript sin necesidad de anotarlos.

### Prueba esto

- Cambia `highlighted` de `true` a `false` en el primer `<ProductCard>` de `App.tsx` — el borde dorado y el fondo amarillo desaparecen; una prop booleana controla dos propiedades de estilo a la vez
- Añade `price?: number` a `ProductCardProps` e imprímela con `{price?.toFixed(2)}` — el optional chaining `?.` evita un error cuando la prop no se pasa
- Cambia el valor por defecto de `description` a `''` (string vacío) y muéstralo con `{description || 'Sin descripción'}` — compara este comportamiento con el de `??` (nullish coalescing)
- Añade `onClick?: () => void` a las props y aplícala al `<div>` con `onClick={onClick}` — TypeScript acepta event handlers opcionales sin wrapper adicional
- Usa `React.CSSProperties` para extraer el estilo del div a una constante: `const cardStyle: React.CSSProperties = { ... }` — necesitarás `import React from 'react'`
- Cambia `borderRadius: 8` a `0` y luego a `20` — observa cómo cambia el aspecto de la tarjeta sin tocar ninguna otra propiedad

---

### `src/components/ProductCatalogList.tsx`

```tsx
// src/components/ProductCatalogList.tsx

interface Product {
  id: number
  name: string
  price: number
  outOfStock?: boolean
}

interface ProductCatalogListProps {
  products: Product[]
  title?: string
}

export default function ProductCatalogList({
  products,
  title = 'Catálogo',
}: ProductCatalogListProps) {
  return (
    <section>
      <h2 style={{ marginBottom: 16 }}>{title}</h2>

      {products.length === 0 && (
        <p style={{ color: '#999' }}>No hay productos disponibles.</p>
      )}

      <ul style={{ listStyle: 'none', padding: 0 }}>
        {products.map((product) => (
          <li
            key={product.id}
            style={{
              display: 'flex',
              justifyContent: 'space-between',
              padding: '10px 0',
              borderBottom: '1px solid #eee',
              opacity: product.outOfStock ? 0.4 : 1,
            }}
          >
            <span>
              {product.name}
              {product.outOfStock && (
                <em style={{ marginLeft: 8, fontSize: 12, color: '#e00' }}>
                  Agotado
                </em>
              )}
            </span>
            <strong>${product.price.toFixed(2)}</strong>
          </li>
        ))}
      </ul>
    </section>
  )
}
```

### Prueba esto

- Pasa `products={[]}` al componente desde `App.tsx` — aparece el mensaje "No hay productos disponibles." gracias al renderizado condicional
- Cambia `outOfStock: true` a `false` en el tercer producto — el elemento vuelve a opacidad completa y el texto "Agotado" desaparece
- Añade `{ id: 5, name: 'Hub USB-C', price: 39.99 }` al array `catalog` — la nueva fila aparece sin cambiar el JSX del componente
- Añade `category?: string` a la interfaz `Product` y muéstrala en un `<em>` dentro del `<span>` — TypeScript marca error si usas la prop antes de declararla
- Cambia `listStyle: 'none'` a `'disc'` en el `<ul>` — los bullets de lista HTML reaparecen; observa que en JSX es `listStyle`, no `list-style`
- Añade un `<footer>` debajo del `<ul>` con `{products.length} producto(s)` — la longitud del array se calcula directamente desde la prop

---

### `src/App.tsx`

```tsx
// src/App.tsx

import ProductCard        from './components/ProductCard'
import ProductCatalogList from './components/ProductCatalogList'

interface Product {
  id: number
  name: string
  price: number
  outOfStock?: boolean
}

const catalog: Product[] = [
  { id: 1, name: 'Teclado mecánico',  price: 89.99 },
  { id: 2, name: 'Monitor 27"',       price: 349.99 },
  { id: 3, name: 'Mouse inalámbrico', price: 29.99, outOfStock: true },
  { id: 4, name: 'Webcam HD',         price: 59.99 },
]

export default function App() {
  return (
    <main style={{ maxWidth: 540, margin: '40px auto', fontFamily: 'sans-serif' }}>

      <ProductCard
        title="Bienvenido a la tienda"
        description="Encuentra los mejores accesorios para tu escritorio"
        highlighted
      />

      <ProductCard title="Oferta del día" description="Webcam HD con 20% de descuento" />

      <ProductCatalogList products={catalog} title="Productos disponibles" />

    </main>
  )
}
```

### Prueba esto

- Añade un tercer `<ProductCard title="Hub USB-C" />` sin `description` — usa el valor por defecto automáticamente
- Intercambia el orden de los dos componentes: pon `<ProductCatalogList>` antes de los `<ProductCard>` — React no impone orden de renderizado
- Pasa `title="Toda la colección"` al `<ProductCatalogList>` — sobreescribe el valor por defecto `'Catálogo'`
- Cambia `maxWidth: 540` en el `<main>` a `800` — el contenedor se amplía sin necesidad de tocar los componentes hijos
- Añade `{ id: 5, name: 'Auriculares BT', price: 79.99 }` al array `catalog` — la lista crece automáticamente

---

## Ejercicio de la página 2

**Objetivo**: practicar JSX, props tipadas, props opcionales y renderizado de listas.

### Enunciado

Crea un componente `UserProfileCard` que reciba las siguientes props:

| Prop | Tipo | Requerida | Descripción |
|---|---|---|---|
| `fullName` | `string` | ✅ | Nombre completo del usuario |
| `email` | `string` | ✅ | Correo electrónico |
| `role` | `'admin' \| 'editor' \| 'viewer'` | ✅ | Rol dentro del sistema |
| `isActive` | `boolean` | ✅ | Si la cuenta está activa |
| `skills` | `string[]` | ✅ | Lista de habilidades |
| `bio` | `string` | ❌ | Descripción breve opcional |

El componente debe:

1. Mostrar `fullName`, `email` y `role`.
2. Si `isActive` es `true`, badge verde **"Activo"**; si no, badge rojo **"Inactivo"**.
3. Mostrar `bio` solo si fue proporcionada.
4. Renderizar `skills` con `.map()`.
5. Usar `interface` para props y `export default` en la declaración.

### Solución de referencia

```tsx
// src/components/UserProfileCard.tsx

interface UserProfileCardProps {
  fullName: string
  email: string
  role: 'admin' | 'editor' | 'viewer'
  isActive: boolean
  skills: string[]
  bio?: string
}

export default function UserProfileCard({
  fullName,
  email,
  role,
  isActive,
  skills,
  bio,
}: UserProfileCardProps) {
  return (
    <div
      style={{
        border: '1px solid #ddd',
        borderRadius: 10,
        padding: 20,
        marginBottom: 16,
        maxWidth: 400,
      }}
    >
      <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
        <h2 style={{ margin: 0 }}>{fullName}</h2>
        <span
          style={{
            backgroundColor: isActive ? '#d4edda' : '#f8d7da',
            color: isActive ? '#155724' : '#721c24',
            padding: '2px 10px',
            borderRadius: 12,
            fontSize: 13,
          }}
        >
          {isActive ? 'Activo' : 'Inactivo'}
        </span>
      </div>

      <p style={{ margin: '8px 0 4px', color: '#555' }}>{email}</p>
      <p style={{ margin: '0 0 12px', fontSize: 13, color: '#888' }}>
        Rol: <strong>{role}</strong>
      </p>

      {bio && <p style={{ fontStyle: 'italic', color: '#444' }}>{bio}</p>}

      <ul style={{ paddingLeft: 18, margin: 0 }}>
        {skills.map((skill) => (
          <li key={skill} style={{ fontSize: 14 }}>{skill}</li>
        ))}
      </ul>
    </div>
  )
}
```

```tsx
// src/App.tsx (para probar el ejercicio)

import UserProfileCard from './components/UserProfileCard'

export default function App() {
  return (
    <main style={{ maxWidth: 480, margin: '40px auto', fontFamily: 'sans-serif' }}>
      <UserProfileCard
        fullName="Ana García"
        email="ana@ejemplo.com"
        role="admin"
        isActive={true}
        skills={['TypeScript', 'React', 'Node.js']}
        bio="Desarrolladora fullstack con 5 años de experiencia."
      />

      <UserProfileCard
        fullName="Luis Mora"
        email="luis@ejemplo.com"
        role="viewer"
        isActive={false}
        skills={['Figma', 'CSS']}
      />
    </main>
  )
}
```

### Prueba esto

- Cambia `isActive={true}` a `false` en la primera `UserProfileCard` — el badge cambia de verde a rojo automáticamente
- Añade `bio="Diseñador con experiencia en sistemas UI"` al segundo `UserProfileCard` (Luis) — el párrafo de bio aparece porque el condicional `{bio && ...}` lo detecta
- Intenta pasar `role="superadmin"` — TypeScript rechaza el valor con un error de tipo; solo acepta los tres literales del tipo unión
- Añade `'GraphQL'` al array `skills` de Ana — la nueva habilidad se renderiza sin cambiar el componente
- Extrae el cálculo de `initials` a una función pura `getInitials(name: string): string` fuera del componente y anota el tipo de retorno
- Añade una prop `avatar?: string` para una URL de imagen y muestra un `<img>` en lugar del círculo de iniciales cuando la prop esté presente

---


---

## Práctica — Componentes básicos tipados sin hooks

Antes de avanzar a props y estado, es fundamental dominar la creación de componentes
simples con TypeScript. Los siguientes ejemplos usan solo JSX, tipos y lógica pura —
sin hooks. Cada uno va en su propio archivo dentro de `src/components/`.

### Estructura de archivos para esta práctica

```
src/
├── components/
│   ├── WelcomeBanner.tsx
│   ├── UserGreeting.tsx
│   ├── CurrentDateDisplay.tsx
│   ├── ColoredBox.tsx
│   ├── ConditionalGreeting.tsx
│   ├── FruitList.tsx
│   ├── PriceTag.tsx
│   ├── StatusBadge.tsx
│   ├── MiniProfileCard.tsx
│   └── SimpleInfoTable.tsx
└── App.tsx
```

---

### `src/components/WelcomeBanner.tsx`

El componente más simple posible: sin props, retorna JSX estático.

```tsx
// src/components/WelcomeBanner.tsx

export default function WelcomeBanner() {
  return (
    <div style={{ background: '#0070f3', color: '#fff', padding: '16px 24px', borderRadius: 8 }}>
      <h1 style={{ margin: 0, fontSize: 22 }}>Bienvenido al curso de React</h1>
      <p style={{ margin: '6px 0 0', opacity: 0.85 }}>Aprende React 19 con TypeScript</p>
    </div>
  )
}
```

### Prueba esto

- Cambia `'#0070f3'` a `'#16a34a'` (verde) como fondo — el banner cambia de color; experimenta con cualquier valor hexadecimal
- Cambia `fontSize: 22` a `32` — el encabezado crece; en estilos inline el número es en píxeles sin necesidad de escribir `'px'`
- Añade `opacity: 0.5` al `<div>` — todo el banner se vuelve semitransparente, incluyendo el texto
- Añade `borderRadius: 0` al `<div>` — el banner pierde sus esquinas redondeadas
- Añade una prop `subtitle?: string` y reemplaza el texto estático del `<p>` con `{subtitle ?? 'Aprende React 19 con TypeScript'}` — el `??` usa el valor por defecto solo cuando `subtitle` es `null` o `undefined`
- Mueve el componente a `src/components/WelcomeBanner.tsx` e impórtalo en `App.tsx` — TypeScript infiere el tipo de retorno del JSX sin anotación explícita

---

### `src/components/UserGreeting.tsx`

Prop `name: string` requerida. Lógica de string dentro del componente.

```tsx
// src/components/UserGreeting.tsx

interface UserGreetingProps {
  name: string
  occupation?: string
}

export default function UserGreeting({ name, occupation }: UserGreetingProps) {
  const initials = name
    .split(' ')
    .map((w) => w[0])
    .join('')
    .toUpperCase()

  return (
    <div style={{ display: 'flex', alignItems: 'center', gap: 12 }}>
      <div
        style={{
          width: 44,
          height: 44,
          borderRadius: '50%',
          background: '#6366f1',
          color: '#fff',
          display: 'flex',
          alignItems: 'center',
          justifyContent: 'center',
          fontWeight: 600,
        }}
      >
        {initials}
      </div>
      <div>
        <p style={{ margin: 0, fontWeight: 600 }}>Hola, {name}</p>
        {occupation && (
          <p style={{ margin: 0, fontSize: 13, color: '#888' }}>{occupation}</p>
        )}
      </div>
    </div>
  )
}
```

```tsx
// Uso
<UserGreeting name="Ana García" occupation="Desarrolladora Frontend" />
<UserGreeting name="Luis Mora" />
```

### Prueba esto

- Pasa `name="Carlos López Ruiz"` — las iniciales calculadas son `"CLR"` con el `.slice(0, 2)` actual; cámbialo a `slice(0, 3)` para ver las tres
- Añade `occupation="DevOps Engineer"` al segundo `<UserGreeting>` — el texto debajo del nombre aparece gracias al `&&` condicional
- Cambia el color del avatar de `'#6366f1'` a `'#e11d48'` — todos los avatares cambian porque el color es interno al componente
- Pasa `name="A"` — el avatar muestra solo `"A"`; el cálculo de iniciales funciona con cualquier longitud de nombre
- Añade `online?: boolean` a las props y renderiza un círculo verde pequeño superpuesto al avatar cuando sea `true` usando `position: 'relative'` en el contenedor e `position: 'absolute'` en el punto
- Cambia `gap: 12` a `gap: 4` en el `<div>` externo — el avatar y el nombre se acercan

---

### `src/components/CurrentDateDisplay.tsx`

Sin props. Usa lógica JavaScript pura — sin estado, el valor se calcula al renderizar.

```tsx
// src/components/CurrentDateDisplay.tsx

export default function CurrentDateDisplay() {
  const now = new Date()

  const fecha = now.toLocaleDateString('es-ES', {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  })

  const hora = now.toLocaleTimeString('es-ES', {
    hour: '2-digit',
    minute: '2-digit',
  })

  return (
    <div style={{ fontSize: 14, color: '#555' }}>
      <span style={{ textTransform: 'capitalize' }}>{fecha}</span>
      <span style={{ marginLeft: 12, color: '#999' }}>{hora}</span>
    </div>
  )
}
```

> Este componente muestra la fecha del momento en que se renderiza.
> Para que se actualice en tiempo real necesitarías `setInterval` + `useState` — eso va en la página 5.

### Prueba esto

- Cambia el locale de `'es-ES'` a `'en-US'` — la fecha y la hora se muestran en formato anglosajón (mes/día/año)
- Añade `second: '2-digit'` al objeto de opciones de `hora` — el display incluye los segundos
- Cambia `weekday: 'long'` a `'short'` — el nombre del día se abrevia (lun., mar., etc.)
- Añade `timeZone: 'America/Mexico_City'` a las opciones de fecha — la hora cambia a la zona horaria de Ciudad de México
- Añade una prop `showTime?: boolean` y condiciona el `<span>` de la hora a que sea `true` con `{showTime && <span>...}</span>}`
- Recarga la página con `Ctrl+R` — la fecha mostrada cambia a la hora actual del nuevo render; sin estado, el valor es fijo por render

---

### `src/components/ColoredBox.tsx`

Props que controlan apariencia. Demuestra estilos dinámicos con TypeScript.

```tsx
// src/components/ColoredBox.tsx

interface ColoredBoxProps {
  color: string
  width?: number
  height?: number
  label?: string
}

export default function ColoredBox({
  color,
  width = 80,
  height = 80,
  label,
}: ColoredBoxProps) {
  return (
    <div style={{ display: 'inline-flex', flexDirection: 'column', alignItems: 'center', gap: 6 }}>
      <div
        style={{
          width,
          height,
          backgroundColor: color,
          borderRadius: 8,
          border: '1px solid rgba(0,0,0,0.1)',
        }}
      />
      {label && <span style={{ fontSize: 12, color: '#666' }}>{label}</span>}
    </div>
  )
}
```

```tsx
// Uso
<ColoredBox color="#0070f3" label="Azul principal" />
<ColoredBox color="#e00" width={60} height={60} label="Peligro" />
<ColoredBox color="#22c55e" label="Éxito" />
```

### Prueba esto

- Cambia los tres colores en `App.tsx` a `'#f59e0b'`, `'#8b5cf6'` y `'#ec4899'` — los cuadros reflejan los nuevos colores sin cambiar el componente
- Pasa `width={120}` y `height={40}` al primer cuadro — la caja se convierte en un rectángulo horizontal
- Quita la prop `label` del tercer cuadro — el `<span>` desaparece gracias al `{label && ...}` condicional
- Añade `borderRadius?: number` con valor por defecto `8` y pásalo al `<div>` — con `borderRadius={50}` y `width === height`, el resultado es un círculo perfecto
- Añade `onClick?: () => void` a las props y aplícalo al `<div>` — en `App.tsx` pasa `onClick={() => alert(color)}` para mostrar el color al hacer clic
- Cambia `border: '1px solid rgba(0,0,0,0.1)'` a `'none'` — el sutil borde desaparece; útil para comparar si el color ya tiene suficiente contraste con el fondo

---

### `src/components/ConditionalGreeting.tsx`

Renderizado condicional con prop booleana y unión de literales.

```tsx
// src/components/ConditionalGreeting.tsx

type TimeOfDay = 'morning' | 'afternoon' | 'evening'

interface ConditionalGreetingProps {
  isLoggedIn: boolean
  userName?: string
  timeOfDay?: TimeOfDay
}

export default function ConditionalGreeting({
  isLoggedIn,
  userName = 'visitante',
  timeOfDay = 'morning',
}: ConditionalGreetingProps) {
  const greetings: Record<TimeOfDay, string> = {
    morning:   'Buenos días',
    afternoon: 'Buenas tardes',
    evening:   'Buenas noches',
  }

  if (!isLoggedIn) {
    return (
      <p style={{ color: '#e00' }}>
        Por favor inicia sesión para continuar.
      </p>
    )
  }

  return (
    <p style={{ color: '#333' }}>
      {greetings[timeOfDay]}, <strong>{userName}</strong>. Bienvenido de vuelta.
    </p>
  )
}
```

```tsx
// Uso
<ConditionalGreeting isLoggedIn={false} />
<ConditionalGreeting isLoggedIn={true} userName="Ana" timeOfDay="afternoon" />
```

### Prueba esto

- Cambia `isLoggedIn={false}` a `true` y añade `userName="Carlos"` — el mensaje de error desaparece y aparece el saludo personalizado
- Prueba cada valor de `timeOfDay`: `'morning'`, `'afternoon'`, `'evening'` — el texto del saludo cambia en cada caso
- Añade un cuarto estado `'night'` al tipo `TimeOfDay` y su entrada en el objeto `greetings` — TypeScript marcará error si el `Record<TimeOfDay, string>` no tiene la clave nueva
- Añade una prop `greeting?: string` que reemplace el saludo del Record cuando esté presente: `greeting ?? greetings[timeOfDay]`
- Cambia el `if (!isLoggedIn)` a un ternario en el `return` — ambas formas son válidas; el early return con `if` suele preferirse para casos de error o loading
- Prueba pasar `timeOfDay` sin valor y observa que usa `'morning'` por el valor por defecto del destructuring

---

### `src/components/FruitList.tsx`

Renderizado de listas tipadas con `.map()`. Key estable con el nombre del item.

```tsx
// src/components/FruitList.tsx

interface Fruit {
  name: string
  emoji: string
  calories: number
}

interface FruitListProps {
  fruits: Fruit[]
  title?: string
}

export default function FruitList({ fruits, title = 'Frutas' }: FruitListProps) {
  if (fruits.length === 0) {
    return <p style={{ color: '#999' }}>No hay frutas en la lista.</p>
  }

  return (
    <div>
      <h3 style={{ marginBottom: 8 }}>{title}</h3>
      <ul style={{ listStyle: 'none', padding: 0 }}>
        {fruits.map((fruit) => (
          <li
            key={fruit.name}
            style={{
              display: 'flex',
              justifyContent: 'space-between',
              padding: '8px 0',
              borderBottom: '1px solid #eee',
            }}
          >
            <span>{fruit.emoji} {fruit.name}</span>
            <span style={{ color: '#888', fontSize: 13 }}>{fruit.calories} kcal</span>
          </li>
        ))}
      </ul>
    </div>
  )
}
```

```tsx
// Uso
const myFruits = [
  { name: 'Manzana', emoji: '🍎', calories: 52 },
  { name: 'Banana',  emoji: '🍌', calories: 89 },
  { name: 'Naranja', emoji: '🍊', calories: 47 },
]

<FruitList fruits={myFruits} title="Frutas y calorías" />
```

### Prueba esto

- Pasa `fruits={[]}` — aparece el mensaje "No hay frutas en la lista." del guard clause antes del `return`
- Añade `{ name: 'Kiwi', emoji: '🥝', calories: 61 }` al array — la nueva fruta aparece sin cambiar el componente
- Añade `inSeason?: boolean` a la interfaz `Fruit` y muestra un `🌟` junto al emoji cuando sea `true`: `{fruit.inSeason && '🌟 '}{fruit.emoji}`
- Cambia la `key` a índice: `.map((fruit, i) => ... key={i})` — funciona pero React lo desaconseja si el orden puede cambiar; `fruit.name` es más estable
- Añade ordenamiento antes del `return`: `const sorted = [...fruits].sort((a, b) => a.calories - b.calories)` y renderiza `sorted` — no mutes `fruits` directamente
- Cambia `borderBottom: '1px solid #eee'` a `borderBottom: 'none'` en el `<li>` y agrega `backgroundColor` alterno usando el índice del `.map()`

---

### `src/components/PriceTag.tsx`

Cálculos con props numéricas. Tipos literales para la moneda.

```tsx
// src/components/PriceTag.tsx

type Currency = 'USD' | 'EUR' | 'COP' | 'MXN'

interface PriceTagProps {
  amount: number
  currency?: Currency
  discountPercent?: number
}

export default function PriceTag({
  amount,
  currency = 'USD',
  discountPercent = 0,
}: PriceTagProps) {
  const hasDiscount = discountPercent > 0
  const finalPrice  = hasDiscount ? amount * (1 - discountPercent / 100) : amount

  const symbols: Record<Currency, string> = {
    USD: '$',
    EUR: '€',
    COP: '$',
    MXN: '$',
  }

  const symbol = symbols[currency]

  return (
    <div style={{ display: 'inline-flex', flexDirection: 'column', alignItems: 'flex-end' }}>
      {hasDiscount && (
        <span style={{ fontSize: 13, color: '#aaa', textDecoration: 'line-through' }}>
          {symbol}{amount.toFixed(2)} {currency}
        </span>
      )}
      <span style={{ fontSize: 20, fontWeight: 700, color: hasDiscount ? '#e00' : '#333' }}>
        {symbol}{finalPrice.toFixed(2)} {currency}
      </span>
      {hasDiscount && (
        <span style={{ fontSize: 12, color: '#22c55e', fontWeight: 500 }}>
          {discountPercent}% de descuento
        </span>
      )}
    </div>
  )
}
```

```tsx
// Uso
<PriceTag amount={99.99} currency="USD" />
<PriceTag amount={99.99} currency="USD" discountPercent={20} />
<PriceTag amount={250000} currency="COP" discountPercent={10} />
```

### Prueba esto

- Cambia `discountPercent={20}` a `50` — el precio final baja a la mitad y el texto del porcentaje se actualiza automáticamente
- Pasa `currency="EUR"` — el símbolo cambia a `€` gracias al `Record<Currency, string>`
- Añade `'GBP'` al tipo `Currency` y su símbolo `'£'` al objeto `symbols` — TypeScript marcará error si el `Record` no tiene la nueva clave
- Quita `discountPercent` o pásalo como `0` — el precio tachado desaparece porque `hasDiscount` es `false`
- Añade `size?: 'small' | 'large'` y cambia el `fontSize` del precio según el tamaño: `16` para `small`, `32` para `large`
- Cambia `color: hasDiscount ? '#e00' : '#333'` a un tercer color cuando el precio sea mayor de 100 usando un ternario anidado

---

### `src/components/StatusBadge.tsx`

Tipo unión para variantes visuales. `Record` para mapear tipos a estilos.

```tsx
// src/components/StatusBadge.tsx

type BadgeStatus = 'active' | 'inactive' | 'pending' | 'error'

interface StatusBadgeProps {
  status: BadgeStatus
  label?: string
}

export default function StatusBadge({ status, label }: StatusBadgeProps) {
  const config: Record<BadgeStatus, { bg: string; color: string; text: string }> = {
    active:   { bg: '#dcfce7', color: '#166534', text: 'Activo' },
    inactive: { bg: '#f3f4f6', color: '#6b7280', text: 'Inactivo' },
    pending:  { bg: '#fef9c3', color: '#854d0e', text: 'Pendiente' },
    error:    { bg: '#fee2e2', color: '#991b1b', text: 'Error' },
  }

  const { bg, color, text } = config[status]

  return (
    <span
      style={{
        backgroundColor: bg,
        color,
        padding: '3px 10px',
        borderRadius: 12,
        fontSize: 12,
        fontWeight: 600,
        display: 'inline-block',
      }}
    >
      {label ?? text}
    </span>
  )
}
```

```tsx
// Uso
<StatusBadge status="active" />
<StatusBadge status="pending" label="En revisión" />
<StatusBadge status="error" />
<StatusBadge status="inactive" />
```

### Prueba esto

- Usa `<StatusBadge status="pending" label="En revisión" />` — la prop `label` sobreescribe el texto por defecto `'Pendiente'` del `config`
- Añade un quinto estado `'warning'` al tipo `BadgeStatus` y al objeto `config` con colores naranjas — TypeScript marcará error hasta que el `Record` esté completo
- Cambia `borderRadius: 12` a `4` — el badge pasa de píldora a rectángulo con esquinas ligeras
- Añade `icon?: string` a las props y renderiza el emoji antes del texto: `` {icon && `${icon} `}{label ?? text} ``
- Cambia `padding: '3px 10px'` a `'6px 16px'` — el badge crece; experimenta con otros valores para ver el efecto del espaciado interno
- Usa `<StatusBadge status="error" />` dentro de `MiniProfileCard` en lugar del badge hard-codeado — observa que los tipos se propagan correctamente entre componentes

---

### `src/components/MiniProfileCard.tsx`

Composición de los conceptos anteriores: props tipadas, opcionales, lógica y renderizado condicional en un componente más completo.

```tsx
// src/components/MiniProfileCard.tsx

import StatusBadge from './StatusBadge'

type BadgeStatus = 'active' | 'inactive' | 'pending' | 'error'

interface MiniProfileCardProps {
  fullName: string
  role: string
  department?: string
  status: BadgeStatus
  joinedYear: number
}

export default function MiniProfileCard({
  fullName,
  role,
  department,
  status,
  joinedYear,
}: MiniProfileCardProps) {
  const initials = fullName
    .split(' ')
    .map((w) => w[0])
    .join('')
    .toUpperCase()
    .slice(0, 2)

  const yearsInCompany = new Date().getFullYear() - joinedYear

  return (
    <div
      style={{
        border: '1px solid #e5e7eb',
        borderRadius: 10,
        padding: 16,
        maxWidth: 280,
        display: 'flex',
        flexDirection: 'column',
        gap: 10,
      }}
    >
      <div style={{ display: 'flex', alignItems: 'center', gap: 12 }}>
        <div
          style={{
            width: 48,
            height: 48,
            borderRadius: '50%',
            background: '#6366f1',
            color: '#fff',
            display: 'flex',
            alignItems: 'center',
            justifyContent: 'center',
            fontWeight: 700,
            fontSize: 16,
            flexShrink: 0,
          }}
        >
          {initials}
        </div>
        <div>
          <p style={{ margin: 0, fontWeight: 600, fontSize: 15 }}>{fullName}</p>
          <p style={{ margin: 0, fontSize: 13, color: '#6b7280' }}>{role}</p>
        </div>
      </div>

      {department && (
        <p style={{ margin: 0, fontSize: 13, color: '#9ca3af' }}>
          📂 {department}
        </p>
      )}

      <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
        <StatusBadge status={status} />
        <span style={{ fontSize: 12, color: '#9ca3af' }}>
          {yearsInCompany === 0
            ? 'Nuevo ingreso'
            : `${yearsInCompany} año${yearsInCompany > 1 ? 's' : ''} en la empresa`}
        </span>
      </div>
    </div>
  )
}
```

```tsx
// Uso
<MiniProfileCard
  fullName="Ana García"
  role="Senior Developer"
  department="Ingeniería"
  status="active"
  joinedYear={2019}
/>
```

### Prueba esto

- Cambia `joinedYear={2019}` a `{new Date().getFullYear()}` — la tarjeta muestra "Nuevo ingreso" porque la diferencia es 0
- Quita la prop `department` — el párrafo del departamento desaparece gracias al `{department && ...}` condicional
- Cambia `status="active"` a `"error"` — el `StatusBadge` interno cambia a rojo; la prop se propaga correctamente
- Cambia el color del avatar de `'#6366f1'` a cualquier color — todas las instancias del componente reflejan el cambio porque está definido internamente
- Añade una segunda `MiniProfileCard` con datos distintos en `App.tsx` — cada instancia calcula sus propias iniciales y su propio contador de años
- Añade `avatarColor?: string` a las props y úsalo en lugar del `'#6366f1'` fijo para personalizar el avatar por instancia

---

### `src/components/SimpleInfoTable.tsx`

Array de objetos con tipo genérico. Encabezados dinámicos desde las keys.

```tsx
// src/components/SimpleInfoTable.tsx

interface TableRow {
  label: string
  value: string | number
  highlight?: boolean
}

interface SimpleInfoTableProps {
  title?: string
  rows: TableRow[]
}

export default function SimpleInfoTable({ title, rows }: SimpleInfoTableProps) {
  return (
    <div style={{ maxWidth: 360 }}>
      {title && <h3 style={{ marginBottom: 8, fontSize: 15 }}>{title}</h3>}
      <table style={{ width: '100%', borderCollapse: 'collapse', fontSize: 14 }}>
        <tbody>
          {rows.map((row) => (
            <tr
              key={row.label}
              style={{
                backgroundColor: row.highlight ? '#fef9c3' : 'transparent',
              }}
            >
              <td
                style={{
                  padding: '8px 12px',
                  borderBottom: '1px solid #e5e7eb',
                  color: '#6b7280',
                  width: '45%',
                }}
              >
                {row.label}
              </td>
              <td
                style={{
                  padding: '8px 12px',
                  borderBottom: '1px solid #e5e7eb',
                  fontWeight: row.highlight ? 600 : 400,
                }}
              >
                {row.value}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  )
}
```

```tsx
// Uso
<SimpleInfoTable
  title="Resumen del pedido"
  rows={[
    { label: 'Subtotal',  value: '$89.99' },
    { label: 'Envío',     value: '$5.00' },
    { label: 'Descuento', value: '-$10.00' },
    { label: 'Total',     value: '$84.99', highlight: true },
  ]}
/>
```

### Prueba esto

- Añade `{ label: 'IVA', value: '$15.20' }` entre "Envío" y "Total" — la nueva fila aparece automáticamente en la tabla
- Cambia `highlight: true` de la fila "Total" a `false` — el fondo amarillo y el `fontWeight: 600` desaparecen
- Agrega `title="Detalle de costos"` en `App.tsx` — el `<h3>` aparece encima de la tabla porque `{title && <h3>...}` lo detecta
- Añade `striped?: boolean` a las props y, cuando sea `true`, alterna el `backgroundColor` de las filas usando el índice del `.map()`: `row_index % 2 === 0 ? '#f9fafb' : 'transparent'`
- Cambia `width: '45%'` de la primera columna `<td>` a `'60%'` — la columna de etiquetas se amplía y la de valores se comprime
- Añade una nueva fila con `value: 0` — observa que el `{row.value}` la muestra sin problema; `0` es un valor válido en JSX a diferencia de `false`, `null` o `undefined`

---

### `src/App.tsx` — navegador de pasos

```tsx
// src/App.tsx
// ┌──────────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.          │
// │   1  WelcomeBanner       — banner estático sin props                    │
// │   2  UserGreeting        — props string + cálculo de iniciales          │
// │   3  CurrentDateDisplay  — fecha calculada al renderizar                │
// │   4  ColoredBox          — estilos dinámicos con props numéricas        │
// │   5  ConditionalGreeting — renderizado condicional + tipo unión         │
// │   6  FruitList           — lista tipada con .map()                      │
// │   7  PriceTag            — cálculos con props numéricas                 │
// │   8  StatusBadge         — Record para mapear tipos a estilos           │
// │   9  MiniProfileCard     — composición de componentes                   │
// │  10  SimpleInfoTable     — tabla con rows tipadas                       │
// │  11  ProductCard         — interfaz de props con opcionales y booleanas │
// │  12  ProductCatalogList  — lista con renderizado condicional de items   │
// │  13  UserProfileCard     — ejercicio: props complejas + rol             │
// └──────────────────────────────────────────────────────────────────────────┘
const PASO = 1

import WelcomeBanner       from './components/WelcomeBanner'
import UserGreeting        from './components/UserGreeting'
import CurrentDateDisplay  from './components/CurrentDateDisplay'
import ColoredBox          from './components/ColoredBox'
import ConditionalGreeting from './components/ConditionalGreeting'
import FruitList           from './components/FruitList'
import PriceTag            from './components/PriceTag'
import StatusBadge         from './components/StatusBadge'
import MiniProfileCard     from './components/MiniProfileCard'
import SimpleInfoTable     from './components/SimpleInfoTable'
import ProductCard         from './components/ProductCard'
import ProductCatalogList  from './components/ProductCatalogList'
import UserProfileCard     from './components/UserProfileCard'

const fruits = [
  { name: 'Manzana', emoji: '🍎', calories: 52 },
  { name: 'Banana',  emoji: '🍌', calories: 89 },
  { name: 'Naranja', emoji: '🍊', calories: 47 },
]

const catalog = [
  { id: 1, name: 'Teclado mecánico',  price: 89.99 },
  { id: 2, name: 'Monitor 27 pulgadas', price: 349.99 },
  { id: 3, name: 'Mouse inalámbrico', price: 29.99, outOfStock: true },
  { id: 4, name: 'Webcam HD',         price: 59.99 },
]

export default function App() {
  const content =
    PASO ===  1 ? <WelcomeBanner /> :
    PASO ===  2 ? <UserGreeting name="Ana García" occupation="Desarrolladora Frontend" /> :
    PASO ===  3 ? <CurrentDateDisplay /> :
    PASO ===  4 ? (
      <div style={{ display: 'flex', gap: 12 }}>
        <ColoredBox color="#0070f3" label="Primary" />
        <ColoredBox color="#22c55e" label="Success" />
        <ColoredBox color="#e00"    label="Danger" />
      </div>
    ) :
    PASO ===  5 ? <ConditionalGreeting isLoggedIn={true} userName="Ana" timeOfDay="afternoon" /> :
    PASO ===  6 ? <FruitList fruits={fruits} title="Frutas favoritas" /> :
    PASO ===  7 ? (
      <div style={{ display: 'flex', gap: 24, alignItems: 'flex-end' }}>
        <PriceTag amount={99.99} currency="USD" />
        <PriceTag amount={99.99} currency="USD" discountPercent={20} />
      </div>
    ) :
    PASO ===  8 ? (
      <div style={{ display: 'flex', gap: 8 }}>
        <StatusBadge status="active" />
        <StatusBadge status="pending" />
        <StatusBadge status="error" />
        <StatusBadge status="inactive" />
      </div>
    ) :
    PASO ===  9 ? (
      <MiniProfileCard
        fullName="Ana García"
        role="Senior Developer"
        department="Ingeniería"
        status="active"
        joinedYear={2019}
      />
    ) :
    PASO === 10 ? (
      <SimpleInfoTable
        title="Resumen del pedido"
        rows={[
          { label: 'Subtotal',  value: '$89.99' },
          { label: 'Envío',     value: '$5.00' },
          { label: 'Total',     value: '$94.99', highlight: true },
        ]}
      />
    ) :
    PASO === 11 ? <ProductCard title="Teclado inalámbrico" description="Bluetooth 5.0, retroiluminado" highlighted /> :
    PASO === 12 ? <ProductCatalogList products={catalog} title="Productos disponibles" /> :
    PASO === 13 ? (
      <UserProfileCard
        fullName="Ana García"
        email="ana@ejemplo.com"
        role="admin"
        isActive={true}
        skills={['TypeScript', 'React', 'Node.js']}
        bio="Desarrolladora fullstack con 5 años de experiencia."
      />
    ) :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <main style={{ maxWidth: 540, margin: '40px auto', fontFamily: 'sans-serif', padding: '0 16px' }}>
      {content}
    </main>
  )
}
```

---

## Ejercicios propuestos — sin hooks

1. **`TemperatureDisplay`** — recibe `celsius: number` y muestra también la conversión a Fahrenheit y Kelvin.
2. **`ProgressBar`** — recibe `percent: number` (0–100) y `color?: string`. Muestra una barra de progreso con el porcentaje visible.
3. **`BusinessCard`** — recibe `name`, `email`, `phone?: string`, `website?: string`. Muestra una tarjeta de presentación. Los campos opcionales solo aparecen si se pasan.
4. **`RatingStars`** — recibe `rating: number` (1–5) y `maxStars?: number`. Muestra estrellas llenas y vacías según el rating.
5. **`TagList`** — recibe `tags: string[]` y `color?: string`. Renderiza cada tag como un badge con `.map()`.

> **Siguiente página →** Funciones como props, eventos tipados, `children` y composición padre ↔ hijo.