# Curso React.js + TypeScript — Página 17
## Frameworks de estilos · Tailwind CSS v4
### LAB: 5 componentes aislados — HOME: landing de cursos con Router

---

## ¿Qué es Tailwind CSS?

Tailwind es un framework **utility-first**: en lugar de escribir clases
semánticas como `.card` o `.hero`, combinas clases atómicas directamente
en el JSX — `rounded-2xl`, `bg-slate-950`, `text-white/60`, `hover:bg-blue-500`.

```tsx
// Sin Tailwind — necesitas un archivo CSS separado
<button className="btn-primary">Enviar</button>

// Con Tailwind — los estilos viven en el JSX
<button className="rounded-xl bg-blue-600 px-4 py-2 font-semibold text-white hover:bg-blue-500 transition">
  Enviar
</button>
```

El resultado: sin archivos CSS extra, sin colisiones de nombres, sin
cambios de contexto entre JSX y CSS.

---

## Tailwind v4 — diferencias clave respecto a v3

| v3 | v4 |
|---|---|
| `tailwind.config.ts` obligatorio | **No existe** — no se crea |
| `@tailwind base/components/utilities` en CSS | Solo `@import "tailwindcss"` |
| Plugin separado `@tailwindcss/vite` era opcional | Plugin oficial requerido |
| Configuración en JS/TS | Configuración en CSS con `@theme` |

---

## Instalación

```bash
npm install tailwindcss @tailwindcss/vite
```

| Paquete | Versión actual |
|---|---|
| `tailwindcss` | 4.2.2 |
| `@tailwindcss/vite` | 4.2.2 |

---

## Configuración — 2 archivos, sin más

### `vite.config.ts`

```ts
// vite.config.ts

import { defineConfig } from 'vite'
import react       from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [react(), tailwindcss()],
  //                 ↑ plugin oficial de Tailwind v4
})
```

### `src/index.css`

```css
/* src/index.css — una sola línea */
@import "tailwindcss";
```

> ⚠️ No crees `tailwind.config.ts` — en v4 no existe.
> Si quieres personalizar colores o fuentes, se hace directamente
> en el CSS con la directiva `@theme`.

---

## Estructura del proyecto

```
src/
├── lab/
│   ├── LabTwButtons.tsx
│   ├── LabTwAlert.tsx
│   ├── LabTwCard.tsx
│   ├── LabTwForm.tsx
│   └── LabTwTable.tsx
├── components/tw/
│   ├── TwNavbar.tsx
│   ├── TwHero.tsx
│   ├── TwCourseGrid.tsx
│   ├── TwNewsletter.tsx
│   └── TwFooter.tsx
├── pages/
│   ├── HomeTW.tsx
│   └── AboutTW.tsx
├── App.tsx         ← alterna entre AppLab y AppHome
├── AppLab.tsx
├── AppHome.tsx
├── index.css       ← solo @import "tailwindcss"
└── main.tsx
```

---

## Fase 1 — Laboratorio

---

### `src/lab/LabTwButtons.tsx`

```tsx
// src/lab/LabTwButtons.tsx

export default function LabTwButtons() {
  return (
    <main className="min-h-screen bg-slate-950 text-white p-8">
      <h2 className="text-xl font-extrabold mb-1">LAB: Buttons</h2>
      <p className="text-white/60 mb-6 text-sm">Variantes, outline, tamaños y estados.</p>

      {/* Variantes */}
      <div className="flex flex-wrap gap-2 mb-4">
        <button className="rounded-xl bg-blue-600 px-4 py-2 font-semibold hover:bg-blue-500 transition">
          Primary
        </button>
        <button className="rounded-xl border border-white/20 px-4 py-2 font-semibold text-white/90 hover:bg-white/10 transition">
          Outline
        </button>
        <button className="rounded-xl bg-emerald-600 px-4 py-2 font-semibold hover:bg-emerald-500 transition">
          Success
        </button>
        <button className="rounded-xl bg-red-600 px-4 py-2 font-semibold hover:bg-red-500 transition">
          Danger
        </button>
        <button className="rounded-xl bg-amber-500 px-4 py-2 font-semibold text-slate-900 hover:bg-amber-400 transition">
          Warning
        </button>
      </div>

      {/* Tamaños */}
      <div className="flex flex-wrap items-center gap-2">
        <button className="rounded-xl bg-blue-600 px-6 py-3 text-lg font-semibold hover:bg-blue-500 transition">
          Grande
        </button>
        <button className="rounded-xl bg-blue-600 px-4 py-2 font-semibold hover:bg-blue-500 transition">
          Normal
        </button>
        <button className="rounded-xl bg-blue-600 px-3 py-1 text-sm font-semibold hover:bg-blue-500 transition">
          Pequeño
        </button>
        <button
          disabled
          className="rounded-xl bg-blue-600 px-4 py-2 font-semibold opacity-40 cursor-not-allowed"
        >
          Deshabilitado
        </button>
      </div>
    </main>
  )
}
```

---

### `src/lab/LabTwAlert.tsx`

```tsx
// src/lab/LabTwAlert.tsx

import { useState } from 'react'

type AlertType = 'success' | 'error' | 'warning' | 'info'

interface AlertProps {
  type:    AlertType
  message: string
  onClose: () => void
}

// Mapa de clases por tipo — objeto tipado en lugar de condicional largo
const STYLES: Record<AlertType, string> = {
  success: 'border-emerald-400/30 bg-emerald-400/10 text-emerald-300',
  error:   'border-red-400/30 bg-red-400/10 text-red-300',
  warning: 'border-amber-400/30 bg-amber-400/10 text-amber-300',
  info:    'border-blue-400/30 bg-blue-400/10 text-blue-300',
}

function Alert({ type, message, onClose }: AlertProps) {
  return (
    <div className={`flex items-start justify-between rounded-2xl border p-4 ${STYLES[type]}`}>
      <span className="text-sm">{message}</span>
      <button
        onClick={onClose}
        className="ml-4 text-current opacity-60 hover:opacity-100 transition text-lg leading-none"
      >
        ✕
      </button>
    </div>
  )
}

export default function LabTwAlert() {
  const [alerts, setAlerts] = useState<AlertType[]>(['success', 'error', 'warning', 'info'])

  const remove = (type: AlertType) =>
    setAlerts(prev => prev.filter(a => a !== type))

  return (
    <main className="min-h-screen bg-slate-950 text-white p-8">
      <h2 className="text-xl font-extrabold mb-1">LAB: Alerts</h2>
      <p className="text-white/60 mb-6 text-sm">Cuatro variantes con cierre individual.</p>

      <div className="flex flex-col gap-3 max-w-xl">
        {alerts.map(type => (
          <Alert
            key={type}
            type={type}
            message={`Alerta de tipo ${type} — haz clic en ✕ para cerrar.`}
            onClose={() => remove(type)}
          />
        ))}
        {alerts.length === 0 && (
          <button
            onClick={() => setAlerts(['success', 'error', 'warning', 'info'])}
            className="self-start rounded-xl border border-white/20 px-4 py-2 text-sm hover:bg-white/10 transition"
          >
            Restaurar alertas
          </button>
        )}
      </div>
    </main>
  )
}
```

---

### `src/lab/LabTwCard.tsx`

```tsx
// src/lab/LabTwCard.tsx

interface CourseCardProps {
  title:    string
  level:    'Básico' | 'Intermedio' | 'Avanzado'
  duration: string
  tag?:     string
}

const LEVEL_COLORS: Record<CourseCardProps['level'], string> = {
  Básico:     'border-emerald-400/30 bg-emerald-400/10 text-emerald-300',
  Intermedio: 'border-amber-400/30 bg-amber-400/10 text-amber-300',
  Avanzado:   'border-red-400/30 bg-red-400/10 text-red-300',
}

function CourseCard({ title, level, duration, tag }: CourseCardProps) {
  return (
    <div className="flex flex-col gap-3 rounded-2xl border border-white/10 bg-white/5 p-5 hover:border-white/20 transition">
      <div className="flex items-center justify-between">
        <span className={`inline-block rounded-full border px-3 py-0.5 text-xs font-semibold ${LEVEL_COLORS[level]}`}>
          {level}
        </span>
        {tag && (
          <span className="rounded-full border border-white/20 bg-white/5 px-3 py-0.5 text-xs text-white/60">
            {tag}
          </span>
        )}
      </div>
      <h3 className="font-bold text-white">{title}</h3>
      <p className="text-sm text-white/60">Duración: {duration}</p>
      <button className="mt-auto self-start rounded-xl border border-white/20 px-4 py-1.5 text-sm font-semibold text-white/80 hover:bg-white/10 transition">
        Ver curso →
      </button>
    </div>
  )
}

export default function LabTwCard() {
  const courses: CourseCardProps[] = [
    { title: 'React + TypeScript', level: 'Básico',     duration: '12h', tag: 'Nuevo'   },
    { title: 'Hooks avanzados',    level: 'Intermedio', duration: '8h',  tag: 'Popular' },
    { title: 'Testing + Vitest',   level: 'Avanzado',   duration: '6h'                  },
  ]

  return (
    <main className="min-h-screen bg-slate-950 text-white p-8">
      <h2 className="text-xl font-extrabold mb-1">LAB: Cards</h2>
      <p className="text-white/60 mb-6 text-sm">Cards tipadas con variante de nivel.</p>
      <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3 max-w-4xl">
        {courses.map(c => <CourseCard key={c.title} {...c} />)}
      </div>
    </main>
  )
}
```

---

### `src/lab/LabTwForm.tsx`

```tsx
// src/lab/LabTwForm.tsx

import { useState } from 'react'

interface FormValues {
  name:  string
  email: string
  role:  string
}

type FormErrors = Partial<Record<keyof FormValues, string>>

export default function LabTwForm() {
  const [values,  setValues]  = useState<FormValues>({ name: '', email: '', role: 'viewer' })
  const [errors,  setErrors]  = useState<FormErrors>({})
  const [success, setSuccess] = useState(false)

  function validate(): boolean {
    const e: FormErrors = {}
    if (!values.name.trim())         e.name  = 'El nombre es requerido'
    if (!values.email.includes('@')) e.email = 'Email inválido'
    setErrors(e)
    return Object.keys(e).length === 0
  }

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    if (!validate()) return
    setSuccess(true)
    setValues({ name: '', email: '', role: 'viewer' })
    setTimeout(() => setSuccess(false), 3000)
  }

  // Función que devuelve clases distintas si el campo tiene error
  const inputClass = (field: keyof FormValues) =>
    `h-11 w-full rounded-xl border bg-white/5 px-4 text-white placeholder:text-white/40 outline-none transition focus:ring-2 ${
      errors[field]
        ? 'border-red-500/60 focus:ring-red-500/30'
        : 'border-white/10 focus:ring-blue-500/40'
    }`

  return (
    <main className="min-h-screen bg-slate-950 text-white p-8">
      <h2 className="text-xl font-extrabold mb-1">LAB: Form</h2>
      <p className="text-white/60 mb-6 text-sm">Con validación y feedback visual.</p>

      {success && (
        <div className="mb-4 rounded-2xl border border-emerald-400/30 bg-emerald-400/10 p-4 text-emerald-300 text-sm max-w-md">
          ✅ Registro enviado correctamente
        </div>
      )}

      <form onSubmit={handleSubmit} className="flex flex-col gap-4 max-w-md">
        <div>
          <label className="mb-1 block text-sm font-semibold text-white/80">Nombre</label>
          <input
            className={inputClass('name')}
            type="text"
            placeholder="Ana García"
            value={values.name}
            onChange={e => {
              setValues(v => ({ ...v, name: e.target.value }))
              setErrors(v => ({ ...v, name: undefined }))
            }}
          />
          {errors.name && <p className="mt-1 text-xs text-red-400">{errors.name}</p>}
        </div>

        <div>
          <label className="mb-1 block text-sm font-semibold text-white/80">Email</label>
          <input
            className={inputClass('email')}
            type="email"
            placeholder="ana@ejemplo.com"
            value={values.email}
            onChange={e => {
              setValues(v => ({ ...v, email: e.target.value }))
              setErrors(v => ({ ...v, email: undefined }))
            }}
          />
          {errors.email && <p className="mt-1 text-xs text-red-400">{errors.email}</p>}
        </div>

        <div>
          <label className="mb-1 block text-sm font-semibold text-white/80">Rol</label>
          <select
            className="h-11 w-full rounded-xl border border-white/10 bg-white/5 px-4 text-white outline-none focus:ring-2 focus:ring-blue-500/40"
            value={values.role}
            onChange={e => setValues(v => ({ ...v, role: e.target.value }))}
          >
            <option value="viewer" className="bg-slate-900">Viewer</option>
            <option value="editor" className="bg-slate-900">Editor</option>
            <option value="admin"  className="bg-slate-900">Admin</option>
          </select>
        </div>

        <button
          type="submit"
          className="h-11 rounded-xl bg-blue-600 font-semibold hover:bg-blue-500 transition"
        >
          Registrar
        </button>
      </form>
    </main>
  )
}
```

---

### `src/lab/LabTwTable.tsx`

```tsx
// src/lab/LabTwTable.tsx

import { useState } from 'react'

interface Product {
  id:       number
  name:     string
  category: string
  price:    number
  active:   boolean
}

const PRODUCTS: Product[] = [
  { id: 1, name: 'Teclado mecánico',  category: 'Periféricos', price: 89.99,  active: true  },
  { id: 2, name: 'Monitor 27"',       category: 'Pantallas',   price: 349.99, active: true  },
  { id: 3, name: 'Mouse inalámbrico', category: 'Periféricos', price: 29.99,  active: false },
  { id: 4, name: 'Webcam HD',         category: 'Cámaras',     price: 59.99,  active: true  },
  { id: 5, name: 'Auriculares BT',    category: 'Audio',       price: 149.99, active: false },
]

export default function LabTwTable() {
  const [search, setSearch] = useState('')

  const filtered = PRODUCTS.filter(p =>
    p.name.toLowerCase().includes(search.toLowerCase()) ||
    p.category.toLowerCase().includes(search.toLowerCase())
  )

  return (
    <main className="min-h-screen bg-slate-950 text-white p-8">
      <h2 className="text-xl font-extrabold mb-1">LAB: Table</h2>
      <p className="text-white/60 mb-4 text-sm">Tabla responsiva con búsqueda en tiempo real.</p>

      <input
        className="mb-4 h-10 w-full max-w-xs rounded-xl border border-white/10 bg-white/5 px-4 text-sm text-white placeholder:text-white/40 outline-none focus:ring-2 focus:ring-blue-500/40"
        placeholder="Buscar producto o categoría..."
        value={search}
        onChange={e => setSearch(e.target.value)}
      />

      <div className="overflow-x-auto rounded-2xl border border-white/10">
        <table className="min-w-full text-sm">
          <thead className="bg-white/5 text-white/70">
            <tr>
              {['#', 'Nombre', 'Categoría', 'Precio', 'Estado'].map(h => (
                <th key={h} className="px-4 py-3 text-left font-semibold">{h}</th>
              ))}
            </tr>
          </thead>
          <tbody>
            {filtered.map(p => (
              <tr key={p.id} className="border-t border-white/10 hover:bg-white/5 transition">
                <td className="px-4 py-3 text-white/50">{p.id}</td>
                <td className="px-4 py-3 font-semibold">{p.name}</td>
                <td className="px-4 py-3 text-white/70">{p.category}</td>
                <td className="px-4 py-3 font-semibold">${p.price.toFixed(2)}</td>
                <td className="px-4 py-3">
                  <span className={`inline-block rounded-full border px-3 py-0.5 text-xs font-semibold ${
                    p.active
                      ? 'border-emerald-400/30 bg-emerald-400/10 text-emerald-300'
                      : 'border-white/10 bg-white/5 text-white/50'
                  }`}>
                    {p.active ? 'Activo' : 'Inactivo'}
                  </span>
                </td>
              </tr>
            ))}
            {filtered.length === 0 && (
              <tr>
                <td colSpan={5} className="px-4 py-6 text-center text-white/40">
                  Sin resultados.
                </td>
              </tr>
            )}
          </tbody>
        </table>
      </div>
    </main>
  )
}
```

---

### `src/AppLab.tsx`

```tsx
// src/AppLab.tsx

import { useState } from 'react'
import LabTwButtons from './lab/LabTwButtons'
import LabTwAlert   from './lab/LabTwAlert'
import LabTwCard    from './lab/LabTwCard'
import LabTwForm    from './lab/LabTwForm'
import LabTwTable   from './lab/LabTwTable'

type LabKey = 'buttons' | 'alert' | 'card' | 'form' | 'table'

export default function AppLab() {
  const [lab, setLab] = useState<LabKey>('buttons')

  return (
    <div className="min-h-screen bg-slate-950">
      <div className="flex items-center gap-4 border-b border-white/10 bg-slate-900 px-4 py-2">
        <span className="font-bold text-white text-sm">Tailwind v4 LAB</span>
        <select
          className="rounded-lg border border-white/10 bg-slate-800 px-3 py-1 text-sm text-white outline-none"
          value={lab}
          onChange={e => setLab(e.target.value as LabKey)}
        >
          <option value="buttons">Buttons</option>
          <option value="alert">Alerts</option>
          <option value="card">Cards</option>
          <option value="form">Form</option>
          <option value="table">Table</option>
        </select>
      </div>

      {lab === 'buttons' && <LabTwButtons />}
      {lab === 'alert'   && <LabTwAlert />}
      {lab === 'card'    && <LabTwCard />}
      {lab === 'form'    && <LabTwForm />}
      {lab === 'table'   && <LabTwTable />}
    </div>
  )
}
```

---

## Fase 2 — Home: landing de cursos con Router

---

### `src/components/tw/TwNavbar.tsx`

```tsx
// src/components/tw/TwNavbar.tsx

import { NavLink } from 'react-router-dom'

// Función que devuelve clases según si la ruta está activa
const linkClass = ({ isActive }: { isActive: boolean }) =>
  `rounded-xl px-3 py-2 text-sm font-semibold transition border ${
    isActive
      ? 'border-white/20 bg-white/10 text-white'
      : 'border-transparent text-white/60 hover:text-white hover:bg-white/10'
  }`

export default function TwNavbar() {
  return (
    <header className="sticky top-0 z-50 border-b border-white/10 bg-slate-950/80 backdrop-blur">
      <div className="mx-auto flex max-w-5xl items-center justify-between px-4 py-3">
        <span className="font-extrabold tracking-tight text-white">DevCursos</span>
        <nav className="flex items-center gap-1">
          <NavLink to="/"      className={linkClass} end>Inicio</NavLink>
          <NavLink to="/about" className={linkClass}>Acerca de</NavLink>
        </nav>
      </div>
    </header>
  )
}
```

> `NavLink` acepta `className` como función — recibe `{ isActive }` y devuelve
> el string de clases. El mismo patrón que `style` en React Router.

---

### `src/components/tw/TwHero.tsx`

```tsx
// src/components/tw/TwHero.tsx

export default function TwHero() {
  return (
    <section className="bg-slate-950 py-16">
      <div className="mx-auto max-w-5xl px-4">
        <span className="inline-block rounded-full border border-blue-400/30 bg-blue-400/10 px-3 py-1 text-xs font-semibold text-blue-300 mb-4">
          Tailwind CSS v4
        </span>
        <h1 className="text-3xl md:text-5xl font-extrabold text-white tracking-tight mb-4">
          Aprende React con<br />
          <span className="text-blue-400">estilos modernos</span>
        </h1>
        <p className="text-white/60 text-lg max-w-xl mb-8">
          Utility-first CSS directamente en tus componentes React.
          Sin archivos CSS externos, sin colisiones de clases.
        </p>
        <div className="flex flex-wrap gap-3">
          <button className="rounded-xl bg-blue-600 px-6 py-3 font-semibold text-white hover:bg-blue-500 transition">
            Ver cursos
          </button>
          <button className="rounded-xl border border-white/20 px-6 py-3 font-semibold text-white/80 hover:bg-white/10 transition">
            Plan de estudio
          </button>
        </div>
      </div>
    </section>
  )
}
```

---

### `src/components/tw/TwCourseGrid.tsx`

```tsx
// src/components/tw/TwCourseGrid.tsx

interface Course {
  id:       number
  title:    string
  level:    'Básico' | 'Intermedio' | 'Avanzado'
  duration: string
  tag?:     string
}

const COURSES: Course[] = [
  { id: 1, title: 'React + TypeScript',   level: 'Básico',     duration: '12h', tag: 'Nuevo'   },
  { id: 2, title: 'Hooks avanzados',      level: 'Intermedio', duration: '8h',  tag: 'Popular' },
  { id: 3, title: 'Testing con Vitest',   level: 'Avanzado',   duration: '6h'                  },
  { id: 4, title: 'TanStack Query',       level: 'Intermedio', duration: '5h',  tag: 'Nuevo'   },
  { id: 5, title: 'React Router v7',      level: 'Básico',     duration: '4h'                  },
  { id: 6, title: 'Performance patterns', level: 'Avanzado',   duration: '7h',  tag: 'Popular' },
]

const LEVEL: Record<Course['level'], string> = {
  Básico:     'border-emerald-400/30 bg-emerald-400/10 text-emerald-300',
  Intermedio: 'border-amber-400/30 bg-amber-400/10 text-amber-300',
  Avanzado:   'border-red-400/30 bg-red-400/10 text-red-300',
}

export default function TwCourseGrid() {
  return (
    <section className="bg-slate-950 py-12">
      <div className="mx-auto max-w-5xl px-4">
        <h2 className="text-2xl font-extrabold text-white mb-2">Cursos disponibles</h2>
        <p className="text-white/50 mb-8">Selecciona el nivel que mejor se adapte a ti.</p>

        <div className="grid gap-4 sm:grid-cols-2 lg:grid-cols-3">
          {COURSES.map(course => (
            <div
              key={course.id}
              className="flex flex-col gap-3 rounded-2xl border border-white/10 bg-white/5 p-5 hover:border-white/20 transition"
            >
              <div className="flex items-center justify-between">
                <span className={`rounded-full border px-3 py-0.5 text-xs font-semibold ${LEVEL[course.level]}`}>
                  {course.level}
                </span>
                {course.tag && (
                  <span className="rounded-full border border-white/15 bg-white/5 px-3 py-0.5 text-xs text-white/50">
                    {course.tag}
                  </span>
                )}
              </div>
              <h3 className="font-bold text-white">{course.title}</h3>
              <p className="text-sm text-white/50">Duración: {course.duration}</p>
              <button className="mt-auto self-start rounded-xl border border-white/20 px-4 py-1.5 text-sm font-semibold text-white/70 hover:bg-white/10 transition">
                Ver curso →
              </button>
            </div>
          ))}
        </div>
      </div>
    </section>
  )
}
```

---

### `src/components/tw/TwNewsletter.tsx`

```tsx
// src/components/tw/TwNewsletter.tsx

import { useState } from 'react'

export default function TwNewsletter() {
  const [email,   setEmail]   = useState('')
  const [success, setSuccess] = useState(false)

  function handleSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault()
    setSuccess(true)
    setEmail('')
    setTimeout(() => setSuccess(false), 3000)
  }

  return (
    <section className="border-t border-white/10 bg-slate-950 py-16">
      <div className="mx-auto max-w-5xl px-4 text-center">
        <h2 className="text-2xl font-extrabold text-white mb-2">Mantente actualizado</h2>
        <p className="text-white/50 mb-8 max-w-md mx-auto">
          Recibe novedades sobre nuevos cursos y contenido exclusivo.
        </p>

        {success && (
          <div className="mb-6 inline-block rounded-2xl border border-emerald-400/30 bg-emerald-400/10 px-6 py-3 text-emerald-300 text-sm">
            ✅ Suscripción registrada (demo)
          </div>
        )}

        <form onSubmit={handleSubmit} className="flex justify-center gap-2 flex-wrap">
          <input
            type="email"
            value={email}
            onChange={e => setEmail(e.target.value)}
            placeholder="tu@correo.com"
            required
            className="h-11 w-72 rounded-xl border border-white/10 bg-white/5 px-4 text-white placeholder:text-white/40 outline-none focus:ring-2 focus:ring-blue-500/40"
          />
          <button
            type="submit"
            className="h-11 rounded-xl bg-blue-600 px-6 font-semibold text-white hover:bg-blue-500 transition"
          >
            Suscribirme
          </button>
        </form>
      </div>
    </section>
  )
}
```

---

### `src/components/tw/TwFooter.tsx`

```tsx
// src/components/tw/TwFooter.tsx

export default function TwFooter() {
  return (
    <footer className="border-t border-white/10 bg-slate-950 py-6">
      <div className="mx-auto flex max-w-5xl flex-wrap items-center justify-between gap-3 px-4">
        <span className="font-extrabold text-white">DevCursos</span>
        <span className="text-sm text-white/40">
          Construido con Tailwind CSS v4 + React 19
        </span>
        <div className="flex gap-4">
          <a href="#" className="text-sm text-white/50 hover:text-white transition">Docs</a>
          <a href="#" className="text-sm text-white/50 hover:text-white transition">GitHub</a>
        </div>
      </div>
    </footer>
  )
}
```

---

### `src/pages/HomeTW.tsx`

```tsx
// src/pages/HomeTW.tsx

import TwHero       from '../components/tw/TwHero'
import TwCourseGrid from '../components/tw/TwCourseGrid'
import TwNewsletter from '../components/tw/TwNewsletter'

export default function HomeTW() {
  return (
    <>
      <TwHero />
      <TwCourseGrid />
      <TwNewsletter />
    </>
  )
}
```

---

### `src/pages/AboutTW.tsx`

```tsx
// src/pages/AboutTW.tsx

export default function AboutTW() {
  return (
    <main className="min-h-screen bg-slate-950 py-16">
      <div className="mx-auto max-w-2xl px-4">
        <h1 className="text-2xl font-extrabold text-white mb-6">Acerca de este proyecto</h1>
        <div className="rounded-2xl border border-white/10 bg-white/5 p-6">
          <ul className="space-y-2 text-white/70 text-sm">
            <li className="flex items-center gap-2">
              <span className="text-blue-400">→</span> React 19 + TypeScript
            </li>
            <li className="flex items-center gap-2">
              <span className="text-blue-400">→</span> Tailwind CSS v4 con plugin de Vite
            </li>
            <li className="flex items-center gap-2">
              <span className="text-blue-400">→</span> React Router v7
            </li>
            <li className="flex items-center gap-2">
              <span className="text-blue-400">→</span> Sin tailwind.config.ts — configuración en CSS
            </li>
          </ul>
        </div>
      </div>
    </main>
  )
}
```

---

### `src/AppHome.tsx`

```tsx
// src/AppHome.tsx

import { BrowserRouter, Routes, Route } from 'react-router-dom'
import TwNavbar from './components/tw/TwNavbar'
import TwFooter from './components/tw/TwFooter'
import HomeTW   from './pages/HomeTW'
import AboutTW  from './pages/AboutTW'

export default function AppHome() {
  return (
    <BrowserRouter>
      <TwNavbar />
      <Routes>
        <Route path="/"      element={<HomeTW />} />
        <Route path="/about" element={<AboutTW />} />
      </Routes>
      <TwFooter />
    </BrowserRouter>
  )
}
```

---

### `src/App.tsx` — alterna entre fases

```tsx
// src/App.tsx

import AppLab from './AppLab'
// import AppHome from './AppHome'

export default function App() {
  return <AppLab />
  // Fase 2 — descomenta y comenta la línea anterior:
  // return <AppHome />
}
```

---

## Clases de Tailwind más usadas — referencia rápida

| Categoría | Ejemplos |
|---|---|
| Colores de fondo | `bg-slate-950`, `bg-white/5`, `bg-blue-600` |
| Colores de texto | `text-white`, `text-white/60`, `text-blue-400` |
| Bordes | `border`, `border-white/10`, `rounded-xl`, `rounded-2xl` |
| Espaciado | `p-4`, `px-6`, `py-3`, `gap-3`, `space-y-2` |
| Flexbox | `flex`, `items-center`, `justify-between`, `flex-wrap` |
| Grid | `grid`, `grid-cols-3`, `sm:grid-cols-2`, `gap-4` |
| Tipografía | `font-bold`, `font-extrabold`, `text-sm`, `text-xl`, `tracking-tight` |
| Hover/transición | `hover:bg-blue-500`, `hover:text-white`, `transition` |
| Responsive | `sm:grid-cols-2`, `md:text-5xl`, `lg:grid-cols-3` |
| Opacidad | `text-white/60`, `bg-white/5`, `border-white/10` |

---

## Resumen de la página 17

- Tailwind v4 no usa `tailwind.config.ts` — la configuración se hace en CSS con `@theme`.
- La instalación mínima es `npm install tailwindcss @tailwindcss/vite`, más el plugin en `vite.config.ts` y `@import "tailwindcss"` en el CSS raíz.
- Las clases se aplican directamente en `className` — no hay archivos CSS separados por componente.
- El patrón `Record<Tipo, string>` para mapear variantes a clases es más limpio que condicionales largos.
- `NavLink` acepta `className` como función `({ isActive }) => string` — permite cambiar clases según la ruta activa sin lógica extra en el componente.
- La opacidad se escribe directamente en la clase de color: `text-white/60` equivale a `color: rgba(255,255,255,0.6)`.
- El grid responsivo se define con prefijos: `grid sm:grid-cols-2 lg:grid-cols-3` sin media queries en línea.

---

> **Siguiente página →** Página 18: Ant Design v6 — LAB + dashboard de ventas.