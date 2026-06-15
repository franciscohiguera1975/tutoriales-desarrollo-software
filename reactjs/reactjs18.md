# Curso React.js + TypeScript — Página 3B
## Módulo 1 · Fundamentos
### Estilos en React: CSS global, inline, CSS Modules, styled-components y theming

---

## Los cuatro enfoques de estilos en React

React no impone ningún sistema de estilos — puedes usar cualquier combinación.
Cada enfoque tiene sus ventajas y casos de uso claros:

| Enfoque | Scope | Soporte `:hover` | Dinámico | Dependencia extra |
|---|---|---|---|---|
| CSS global | Global — puede colisionar | ✅ | Manual | ❌ |
| Inline styles | Por elemento | ❌ | ✅ directo | ❌ |
| CSS Modules | Local — sin colisión | ✅ | Manual | ❌ |
| styled-components | Por componente | ✅ | ✅ por props | ✅ `styled-components` |

---

## Instalación

Solo `styled-components` requiere instalación extra.
En v6 **los tipos ya están incluidos** — no necesitas `@types/styled-components`:

```bash
npm install styled-components
# En v6 NO instales @types/styled-components — viene incluido
```

---

## Estructura del proyecto

```
src/
├── styles/
│   ├── global.css          ← CSS global
│   └── card.module.css     ← CSS Modules
├── theme/
│   ├── theme.css           ← CSS variables light/dark
│   └── ThemeContext.tsx    ← Context para el tema
├── hooks/
│   ├── useStyles.ts        ← hook para cambiar estilos en tiempo real
│   └── useHover.ts         ← hook para estilos al pasar el cursor
├── components/
│   ├── CssGlobalDemo.tsx
│   ├── InlineStyleDemo.tsx
│   ├── CssModuleDemo.tsx
│   ├── StyledComponentsDemo.tsx
│   ├── LiveStyleEditor.tsx
│   ├── HoverDemo.tsx
│   └── ThemePanel.tsx
├── App.tsx
└── main.tsx
```

---

## 1 · CSS Global

El enfoque más simple: un archivo `.css` que se importa directamente.
Las clases son globales — cualquier componente que importe el archivo
puede usarlas, pero también pueden colisionar con clases de otros archivos.

### `src/styles/global.css`

```css
/* src/styles/global.css */

.globalCard {
  border:        1px solid var(--border);
  border-radius: 10px;
  padding:       16px;
}

.globalTitle {
  margin:      0 0 8px 0;
  color:       var(--accent);
  font-weight: 800;
}
```

### `src/components/CssGlobalDemo.tsx`

```tsx
// src/components/CssGlobalDemo.tsx

import '../styles/global.css'

export default function CssGlobalDemo() {
  return (
    <div className="globalCard">
      <h3 className="globalTitle">CSS Global</h3>
      <p style={{ margin: 0, color: 'var(--muted)' }}>
        Clases definidas en un archivo <code>.css</code> importado en el componente.
        Scope global — pueden colisionar si dos componentes usan el mismo nombre de clase.
      </p>
    </div>
  )
}
```

### Prueba esto

- Cambia `.globalCard { border-radius: 10px }` a `border-radius: 0` en `global.css` — guarda y observa cómo el cambio afecta a todos los elementos que usen esa clase en la app
- Renombra la clase `.globalTitle` a `.title` en el CSS y en el componente — observa que si otro componente ya tiene una clase `.title`, los estilos colisionarían en el DOM
- Añade una segunda clase `.globalSubtitle { color: red; font-size: 12px }` en el archivo CSS e inyéctala en el componente con `className="globalSubtitle"` en el párrafo
- Importa `global.css` desde dos componentes distintos — abre DevTools y comprueba que la hoja de estilos se incluye una sola vez en el DOM
- Añade `font-family: monospace` a `.globalCard` — el cambio aplica globalmente a todos los componentes que usen esa clase sin tocar el JSX

> **Cuándo usarlo**: estilos base, reset CSS, tipografía global, variables de diseño.
> Para componentes individuales, CSS Modules es más seguro.

---

## 2 · Inline styles

Los estilos van como objetos JavaScript en la prop `style`.
TypeScript valida el objeto contra `React.CSSProperties` — cualquier propiedad
mal escrita o con valor incorrecto da error en el editor.

### `src/components/InlineStyleDemo.tsx`

```tsx
// src/components/InlineStyleDemo.tsx

import type { CSSProperties } from 'react'

export default function InlineStyleDemo() {
  // CSSProperties tipa el objeto — TypeScript detecta errores al escribir
  const card: CSSProperties = {
    border:       '1px solid var(--border)',
    background:   'var(--card)',
    borderRadius: 10,
    padding:      16,
  }

  const title: CSSProperties = {
    margin:     '0 0 8px 0',
    color:      'var(--accent)',
    fontWeight: 800,
  }

  return (
    <div style={card}>
      <h3 style={title}>Inline styles</h3>
      <p style={{ margin: 0, color: 'var(--muted)' }}>
        Estilos como objetos JS dentro del componente. Útil para valores dinámicos
        pero sin soporte de pseudo-clases (<code>:hover</code>) ni media queries.
      </p>
    </div>
  )
}
```

### Prueba esto

- Añade `boxShadow: '0 2px 8px rgba(0,0,0,0.1)'` al objeto `card` — guarda y observa la sombra aplicada de inmediato sin tocar ningún archivo CSS
- Escribe `backround: 'red'` (con typo) en el objeto `card` — TypeScript muestra un error en el editor porque `backround` no es una propiedad válida de `CSSProperties`
- Añade `:hover` directamente dentro del objeto de estilos como `':hover': { background: 'blue' }` — observa que no tiene efecto; los inline styles no soportan pseudo-clases
- Cambia `fontWeight: 800` a `fontWeight: 'extrabold'` — TypeScript reporta error porque el valor esperado es un número o una de las cadenas válidas como `'bold'`
- Convierte el objeto `title` en un estado con `useState` e incluye un botón que lo modifique en tiempo real — observa cómo el re-render aplica los nuevos estilos instantáneamente

> **Limitación**: `style` no soporta `:hover`, `:focus`, `@media` ni animaciones CSS.
> Para esos casos usa CSS Modules o styled-components.

---

## 3 · CSS Modules

Vite soporta CSS Modules sin configuración extra — cualquier archivo que termine
en `.module.css` activa el scope local automáticamente. Cada clase recibe un
nombre único generado en build time: `.btn` puede convertirse en `.btn_a3f9k`.

### `src/styles/card.module.css`

```css
/* src/styles/card.module.css */

.card {
  border:        1px solid var(--border);
  background:    var(--card);
  border-radius: 10px;
  padding:       16px;
}

.title {
  margin:      0 0 8px 0;
  color:       var(--accent);
  font-weight: 800;
}

.btn {
  padding:       8px 16px;
  background:    var(--accent);
  color:         white;
  border:        none;
  border-radius: 8px;
  cursor:        pointer;
  font-weight:   600;
}

.btn:hover {
  filter: brightness(1.1);   /* :hover sí funciona en CSS Modules */
}
```

### `src/components/CssModuleDemo.tsx`

```tsx
// src/components/CssModuleDemo.tsx

import styles from '../styles/card.module.css'

export default function CssModuleDemo() {
  return (
    <div className={styles.card}>
      <h3 className={styles.title}>CSS Modules</h3>
      <p style={{ margin: '0 0 12px', color: 'var(--muted)' }}>
        Cada clase recibe un nombre único generado en build time.
        Elimina colisiones sin necesitar BEM ni prefijos manuales.
      </p>
      <button className={styles.btn}>Botón con módulo</button>
    </div>
  )
}
```

### Prueba esto

- Pasa el cursor sobre el botón — observa el efecto `brightness(1.1)` del selector `.btn:hover` que funciona en CSS Modules pero no en inline styles
- Escribe `styles.boton` en lugar de `styles.btn` — TypeScript muestra un error inmediato porque la clase `boton` no existe en el módulo
- Abre DevTools y examina el atributo `class` del botón — verás un nombre generado como `_btn_a3f9k` en lugar del `.btn` original, confirmando el scope local
- Añade una clase `.highlight { background: yellow }` en `card.module.css` y úsala con `className={styles.highlight}` — funciona sin riesgo de colisión con otras clases `.highlight` del proyecto
- Añade `@media (max-width: 480px) { .card { padding: 8px } }` en el módulo CSS — reduce la ventana del navegador y observa que el padding cambia solo en pantallas pequeñas

> `styles` es un objeto TypeScript — si escribes `styles.btnTypo` y la clase
> no existe, TypeScript lo detecta como error.

---

## 4 · styled-components v6

CSS-in-JS: escribes CSS dentro de TypeScript, vinculado directamente al componente.
El scope es automático, soporta `:hover`, media queries y recibe props tipadas.

**Novedad de v6**: las props que controlan estilos deben usar el prefijo `$`
(props transient) para evitar que pasen al DOM y generen warnings:

```tsx
// ❌ v5 — la prop variant llega al DOM como atributo HTML
const Btn = styled.button<{ variant: 'primary' | 'outline' }>``

// ✅ v6 — prefijo $ indica que es solo para estilos, no llega al DOM
const Btn = styled.button<{ $variant?: 'primary' | 'outline' }>``
```

### `src/components/StyledComponentsDemo.tsx`

```tsx
// src/components/StyledComponentsDemo.tsx

import styled from 'styled-components'

// Props transient con prefijo $ — no pasan al DOM en v6
interface BtnProps {
  $variant?: 'primary' | 'outline'
}

const Card = styled.div`
  border:        1px solid var(--border);
  background:    var(--card);
  border-radius: 10px;
  padding:       16px;
`

const Title = styled.h3`
  margin:      0 0 8px 0;
  color:       var(--accent);
  font-weight: 800;
`

const Btn = styled.button<BtnProps>`
  padding:       8px 16px;
  border-radius: 8px;
  cursor:        pointer;
  font-weight:   600;
  border:        1px solid var(--accent);
  background:    ${p => p.$variant === 'outline' ? 'transparent' : 'var(--accent)'};
  color:         ${p => p.$variant === 'outline' ? 'var(--accent)' : 'white'};
  transition:    filter 0.15s;

  &:hover {
    filter: brightness(1.1);
  }
`

export default function StyledComponentsDemo() {
  return (
    <Card>
      <Title>Styled-components v6</Title>
      <p style={{ margin: '0 0 12px', color: 'var(--muted)' }}>
        CSS-in-JS con scope automático. Props transient con prefijo <code>$</code>
        en v6 para no contaminar el DOM.
      </p>
      <div style={{ display: 'flex', gap: 8 }}>
        <Btn>Primary</Btn>
        <Btn $variant="outline">Outline</Btn>
      </div>
    </Card>
  )
}
```

### Prueba esto

- Cambia `$variant="outline"` a `$variant="primary"` — ambos botones renderizan con fondo de color; confirma que la lógica de prop controla el estilo
- Elimina el prefijo `$` de la interfaz y del uso: `variant` en lugar de `$variant` — abre DevTools y observa el warning de React sobre un atributo desconocido en el DOM
- Añade una tercera variante `$variant?: 'primary' | 'outline' | 'danger'` y agrega `background: ${p => p.$variant === 'danger' ? '#dc2626' : ...}` — el botón cambia de color según la prop
- Añade `&:active { transform: scale(0.97) }` dentro del template literal de `Btn` — haz clic en el botón y observa el efecto de escala en la interacción
- Añade `font-size: 18px` directamente en el template literal de `Card` — el cambio aplica solo a ese componente sin afectar otras tarjetas del proyecto

---

## 5 · Hooks para estilos dinámicos en tiempo real

Los hooks permiten controlar estilos como si fueran estado — cambian
la UI sin necesidad de clases adicionales ni lógica duplicada.

### `src/hooks/useStyles.ts`

Hook que expone funciones para cambiar color, tamaño y peso tipográfico
de cualquier elemento desde controles de formulario:

```ts
// src/hooks/useStyles.ts

import { useState, useCallback } from 'react'
import type { CSSProperties } from 'react'

interface UseStylesReturn {
  style:    CSSProperties
  setColor: (color: string) => void
  setSize:  (size: number) => void
  setBold:  (bold: boolean) => void
  reset:    () => void
}

const DEFAULT: CSSProperties = {
  color:      '#111827',
  fontSize:   16,
  fontWeight: 400,
}

export function useStyles(
  initial: CSSProperties = DEFAULT
): UseStylesReturn {
  const [style, setStyle] = useState<CSSProperties>(initial)

  const setColor = useCallback((color: string) => {
    setStyle(prev => ({ ...prev, color }))
  }, [])

  const setSize = useCallback((size: number) => {
    setStyle(prev => ({ ...prev, fontSize: size }))
  }, [])

  const setBold = useCallback((bold: boolean) => {
    setStyle(prev => ({ ...prev, fontWeight: bold ? 700 : 400 }))
  }, [])

  const reset = useCallback(() => setStyle(initial), [initial])

  return { style, setColor, setSize, setBold, reset }
}
```

### `src/components/LiveStyleEditor.tsx`

```tsx
// src/components/LiveStyleEditor.tsx

import { useStyles } from '../hooks/useStyles'

export default function LiveStyleEditor() {
  const { style, setColor, setSize, setBold, reset } = useStyles({
    color:      '#111827',
    fontSize:   16,
    fontWeight: 400,
  })

  return (
    <div style={{
      border:       '1px solid var(--border)',
      background:   'var(--card)',
      borderRadius: 10,
      padding:      16,
    }}>
      <h3 style={{ margin: '0 0 12px', color: 'var(--accent)', fontWeight: 800 }}>
        Hook useStyles — editor en tiempo real
      </h3>

      {/* Controles */}
      <div style={{ display: 'flex', flexWrap: 'wrap', gap: 12, marginBottom: 16 }}>
        <label style={{ display: 'flex', flexDirection: 'column', gap: 4, fontSize: 13, color: 'var(--muted)' }}>
          Color
          <input
            type="color"
            defaultValue="#111827"
            onChange={e => setColor(e.target.value)}
            style={{ width: 48, height: 32, border: 'none', cursor: 'pointer' }}
          />
        </label>

        <label style={{ display: 'flex', flexDirection: 'column', gap: 4, fontSize: 13, color: 'var(--muted)' }}>
          Tamaño
          <input
            type="range"
            min={12}
            max={36}
            defaultValue={16}
            onChange={e => setSize(Number(e.target.value))}
          />
        </label>

        <label style={{ display: 'flex', alignItems: 'center', gap: 6, fontSize: 13, color: 'var(--muted)', cursor: 'pointer' }}>
          <input
            type="checkbox"
            onChange={e => setBold(e.target.checked)}
          />
          Negrita
        </label>

        <button
          onClick={reset}
          style={{
            padding:      '4px 12px',
            border:       '1px solid var(--border)',
            borderRadius: 6,
            background:   'transparent',
            color:        'var(--muted)',
            cursor:       'pointer',
            fontSize:     13,
            alignSelf:    'flex-end',
          }}
        >
          Reset
        </button>
      </div>

      {/* Preview en tiempo real */}
      <div style={{
        padding:      12,
        border:       '1px dashed var(--border)',
        borderRadius: 8,
        background:   'var(--bg)',
      }}>
        <p style={{ margin: 0, ...style }}>
          Este texto cambia de estilo en tiempo real usando el hook useStyles.
        </p>
      </div>
    </div>
  )
}
```

### Prueba esto

- Mueve el slider de tamaño hasta el máximo — observa el texto crecer a 36px en tiempo real sin recargar la página
- Activa "Negrita" y luego haz clic en "Reset" — el estilo vuelve exactamente al estado inicial definido en `DEFAULT`
- Cambia el color con el picker y luego activa "Negrita" — el color se mantiene porque `setBold` usa el operador spread `{ ...prev, fontWeight }` en lugar de reemplazar el estado completo
- Añade una función `setItalic` al hook siguiendo el mismo patrón que `setBold` — agrega un checkbox en `LiveStyleEditor` para activar/desactivar la cursiva
- Modifica `DEFAULT` para que `fontSize` sea `24` en lugar de `16` — al montar el componente el texto aparece con 24px; haz clic en "Reset" y confirma que vuelve a 24px

---

### `src/hooks/useHover.ts`

Encapsula los event handlers `onMouseEnter`/`onMouseLeave` y la lógica
de mezclar estilos base con estilos de hover:

```ts
// src/hooks/useHover.ts

import { useState, useCallback } from 'react'
import type { CSSProperties } from 'react'

interface UseHoverReturn {
  hoverProps: {
    onMouseEnter: () => void
    onMouseLeave: () => void
  }
  style: CSSProperties
}

export function useHover(
  baseStyle:  CSSProperties,
  hoverStyle: CSSProperties
): UseHoverReturn {
  const [hovered, setHovered] = useState(false)

  const onMouseEnter = useCallback(() => setHovered(true),  [])
  const onMouseLeave = useCallback(() => setHovered(false), [])

  return {
    hoverProps: { onMouseEnter, onMouseLeave },
    style:      hovered ? { ...baseStyle, ...hoverStyle } : baseStyle,
  }
}
```

### `src/components/HoverDemo.tsx`

```tsx
// src/components/HoverDemo.tsx

import { useHover } from '../hooks/useHover'

export default function HoverDemo() {
  const btn1 = useHover(
    {
      padding: '10px 20px', background: '#0070f3', color: 'white',
      border: 'none', borderRadius: 8, cursor: 'pointer',
      fontWeight: 600, transition: 'all 0.2s',
    },
    {
      background: '#2563eb',
      transform:  'translateY(-2px)',
      boxShadow:  '0 4px 12px rgba(0,112,243,0.35)',
    }
  )

  const btn2 = useHover(
    {
      padding: '10px 20px', background: 'transparent', color: 'var(--accent)',
      border: '1px solid var(--accent)', borderRadius: 8, cursor: 'pointer',
      fontWeight: 600, transition: 'all 0.2s',
    },
    { background: 'var(--accent)', color: 'white' }
  )

  const card = useHover(
    {
      border: '1px solid var(--border)', background: 'var(--card)',
      borderRadius: 10, padding: 16, transition: 'all 0.2s', cursor: 'default',
    },
    {
      borderColor: 'var(--accent)',
      boxShadow:   '0 4px 16px rgba(0,0,0,0.08)',
    }
  )

  return (
    <div {...card.hoverProps} style={card.style}>
      <h3 style={{ margin: '0 0 12px', color: 'var(--accent)', fontWeight: 800 }}>
        Hook useHover — estilos al pasar el cursor
      </h3>
      <p style={{ margin: '0 0 14px', color: 'var(--muted)', fontSize: 14 }}>
        Pasa el cursor sobre la tarjeta y sobre los botones para ver los efectos.
        El hook devuelve <code>hoverProps</code> y el <code>style</code> mezclado.
      </p>
      <div style={{ display: 'flex', gap: 10 }}>
        <button {...btn1.hoverProps} style={btn1.style}>Hover elevación</button>
        <button {...btn2.hoverProps} style={btn2.style}>Hover relleno</button>
      </div>
    </div>
  )
}
```

### Prueba esto

- Pasa el cursor sobre el botón "Hover elevación" — observa la sombra y el `translateY(-2px)` que lo eleva visualmente; aleja el cursor y vuelve al estado base
- Pasa el cursor sobre la tarjeta — el borde cambia a `var(--accent)` y aparece la sombra; el hook `useHover` en `card` aplica el mismo patrón que en los botones
- Cambia `transform: 'translateY(-2px)'` a `transform: 'translateY(-6px)'` en `btn1` — el efecto de elevación es más pronunciado al hacer hover
- Añade `opacity: 0.8` al `hoverStyle` de `btn2` — el botón se vuelve ligeramente transparente además de cambiar de color al pasar el cursor
- Cambia `transition: 'all 0.2s'` a `transition: 'all 1s'` en el `baseStyle` de `btn1` — el efecto de hover aplica en cámara lenta durante 1 segundo

---

## 6 · Theming con Context + CSS variables

El patrón más robusto para tema claro/oscuro: las CSS variables se definen
en un archivo `.css` con dos bloques (`:root` y `[data-theme="dark"]`),
y un Context de React gestiona qué bloque está activo.

### `src/theme/theme.css`

```css
/* src/theme/theme.css */

:root {
  --bg:     #ffffff;
  --card:   #f9fafb;
  --border: #e5e7eb;
  --text:   #111827;
  --accent: #0070f3;
  --muted:  #6b7280;
}

[data-theme="dark"] {
  --bg:     #0f172a;
  --card:   #1e293b;
  --border: #334155;
  --text:   #f1f5f9;
  --accent: #60a5fa;
  --muted:  #94a3b8;
}
```

### `src/theme/ThemeContext.tsx`

```tsx
// src/theme/ThemeContext.tsx
// React 19 — <ThemeContext value={...}> sin .Provider

import { createContext, useContext, useState } from 'react'

type Theme = 'light' | 'dark'

interface ThemeContextValue {
  theme:       Theme
  toggleTheme: () => void
}

const ThemeContext = createContext<ThemeContextValue | null>(null)

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('light')

  function toggleTheme() {
    setTheme(t => t === 'light' ? 'dark' : 'light')
  }

  return (
    <ThemeContext value={{ theme, toggleTheme }}>
      <div
        data-theme={theme}
        style={{
          background: 'var(--bg)',
          color:      'var(--text)',
          minHeight:  '100vh',
          transition: 'background 0.25s, color 0.25s',
        }}
      >
        {children}
      </div>
    </ThemeContext>
  )
}

export function useTheme(): ThemeContextValue {
  const ctx = useContext(ThemeContext)
  if (!ctx) throw new Error('useTheme debe usarse dentro de ThemeProvider')
  return ctx
}
```

### `src/components/ThemePanel.tsx`

```tsx
// src/components/ThemePanel.tsx

import { useTheme } from '../theme/ThemeContext'

export default function ThemePanel() {
  const { theme, toggleTheme } = useTheme()

  return (
    <div style={{
      border:       '1px solid var(--border)',
      background:   'var(--card)',
      borderRadius: 10,
      padding:      16,
    }}>
      <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center', marginBottom: 12 }}>
        <h3 style={{ margin: 0, color: 'var(--accent)', fontWeight: 800 }}>
          Theming con Context + CSS variables
        </h3>
        <button
          onClick={toggleTheme}
          style={{
            padding:      '6px 14px',
            border:       '1px solid var(--border)',
            borderRadius: 8,
            background:   'var(--bg)',
            color:        'var(--text)',
            cursor:       'pointer',
            fontWeight:   600,
            fontSize:     13,
          }}
        >
          {theme === 'light' ? '🌙 Modo oscuro' : '☀️ Modo claro'}
        </button>
      </div>

      <p style={{ margin: '0 0 12px', color: 'var(--muted)', fontSize: 14 }}>
        El atributo <code>data-theme</code> en el contenedor raíz activa el bloque
        CSS correspondiente. Todos los componentes heredan las variables sin
        necesidad de props ni contexto adicional.
      </p>

      {/* Paleta visual de las variables activas */}
      <div style={{ display: 'flex', gap: 8, flexWrap: 'wrap' }}>
        {['--bg', '--card', '--border', '--text', '--accent', '--muted'].map(v => (
          <div
            key={v}
            style={{
              padding:      '4px 10px',
              background:   `var(${v})`,
              border:       '1px solid var(--border)',
              borderRadius: 6,
              fontSize:     12,
              color:        v === '--bg' || v === '--card' ? 'var(--text)' : 'var(--bg)',
            }}
          >
            {v}
          </div>
        ))}
      </div>
    </div>
  )
}
```

---

## Navegador de pasos — `App.tsx`

```tsx
// src/App.tsx

import { ThemeProvider }    from './theme/ThemeContext'
import CssGlobalDemo        from './components/CssGlobalDemo'
import InlineStyleDemo      from './components/InlineStyleDemo'
import CssModuleDemo        from './components/CssModuleDemo'
import StyledComponentsDemo from './components/StyledComponentsDemo'
import LiveStyleEditor      from './components/LiveStyleEditor'
import HoverDemo            from './components/HoverDemo'
import ThemePanel           from './components/ThemePanel'
import './theme/theme.css'

// ┌──────────────────────────────────────────────────────────────────────┐
// │  Cambia PASO y guarda (Ctrl+S) para navegar entre componentes.      │
// │  1  CssGlobalDemo        — clases globales y riesgo de colisión     │
// │  2  InlineStyleDemo      — objetos JS, sin :hover ni @media         │
// │  3  CssModuleDemo        — scope local, :hover con CSS Modules      │
// │  4  StyledComponentsDemo — CSS-in-JS con props transient ($)        │
// │  5  LiveStyleEditor      — hook useStyles para estilos dinámicos    │
// │  6  HoverDemo            — hook useHover para efectos hover         │
// │  7  ThemePanel           — Context + CSS variables para theming     │
// └──────────────────────────────────────────────────────────────────────┘
const PASO = 1

export default function App() {
  const content =
    PASO === 1 ? <CssGlobalDemo /> :
    PASO === 2 ? <InlineStyleDemo /> :
    PASO === 3 ? <CssModuleDemo /> :
    PASO === 4 ? <StyledComponentsDemo /> :
    PASO === 5 ? <LiveStyleEditor /> :
    PASO === 6 ? <HoverDemo /> :
    PASO === 7 ? <ThemePanel /> :
    <p style={{ color: '#e00' }}>Paso {PASO}: crea el componente primero</p>

  return (
    <ThemeProvider>
      <main style={{ maxWidth: 640, margin: '0 auto', padding: '32px 16px' }}>
        {content}
      </main>
    </ThemeProvider>
  )
}
```

### Prueba esto

- Cambia `PASO = 1` y guarda — ve clases globales aplicadas con `className`
- Cambia a `PASO = 2` — observa que los objetos JS de estilo aceptan camelCase
- Cambia a `PASO = 3` — inspecciona en DevTools el nombre `Button_button__XXXXX` generado por CSS Modules
- Cambia a `PASO = 4` — modifica props `$variant` o `$size` para ver cómo cambia el estilo con styled-components
- Cambia a `PASO = 5` — ajusta los controles del `LiveStyleEditor` y ve los cambios en tiempo real
- Cambia a `PASO = 6` — pasa el cursor sobre el elemento y verifica el estado hover via `useHover`
- Cambia a `PASO = 7` — activa el toggle de tema oscuro y observa cómo `ThemePanel` lee las CSS variables vía Context

---

## Cuándo usar cada enfoque

| Situación | Enfoque recomendado |
|---|---|
| Estilos globales, reset, variables | CSS global |
| Estilo calculado en runtime con JS | Inline styles |
| Componentes con estilos propios y `:hover` | CSS Modules |
| Componentes con muchas variantes por props | styled-components |
| Tema claro/oscuro global | Context + CSS variables |
| Cambiar estilos desde controles de UI | Hook `useStyles` |

---

## Resumen de la página 3B

- CSS global: rápido y simple, riesgo de colisión de nombres entre componentes.
- Inline styles: totalmente dinámico pero sin `:hover`, `:focus` ni `@media`.
- CSS Modules: nombres únicos automáticos, soporta `:hover` — el equilibrio para la mayoría de los casos.
- styled-components v6: CSS-in-JS con scope, props tipadas y soporte completo de CSS. No instalar `@types/styled-components` — los tipos vienen incluidos. Props de estilo usan prefijo `$`.
- CSS variables en `:root` y `[data-theme="dark"]` — el patrón más eficiente para theming. Los componentes leen las variables sin saber si el tema es claro u oscuro.
- `useStyles` y `useHover` — encapsulan la lógica de cambio de estilos igual que cualquier otro hook de estado.

---

> **Siguiente página →** Página 4: `useState` en profundidad.
> **Frameworks de estilos →** Páginas 16–19: React-Bootstrap, Tailwind CSS, Ant Design y Material UI.