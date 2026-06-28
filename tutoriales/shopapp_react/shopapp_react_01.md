# ShopApp React Web — Módulo 1

## Setup — Vite, TypeScript, Tailwind CSS, shadcn/ui y Clean Architecture

**Duración estimada:** 2–3 horas
**Prerequisitos:** Node.js ≥ 18, npm ≥ 9, backend `shopapi_01` desplegado.

---

> **Objetivo**
> Crear el proyecto React con Vite y TypeScript, instalar todas las dependencias del tutorial,
> configurar Tailwind CSS y shadcn/ui, y establecer la estructura de Clean Architecture de capas
> globales que se usará en todos los módulos siguientes.
>
> **Checkpoint final**
> - `npm run dev` levanta la app en `http://localhost:5173` sin errores de compilación.
> - La estructura de carpetas refleja Clean Architecture (`data/`, `domain/`, `presentation/`).
> - React Router v6 muestra una pantalla de bienvenida en `/`.
> - Tailwind y shadcn/ui están configurados y un componente `Button` de shadcn renderiza correctamente.

---

## 1.1 Crear el proyecto con Vite

```bash
npm create vite@latest shopapp-react -- --template react-ts
cd shopapp-react
npm install
```

Verifica que el proyecto compila:

```bash
npm run dev
```

Deberías ver la pantalla por defecto de Vite en `http://localhost:5173`. Detén el servidor con `Ctrl+C`.

---

## 1.2 Instalar dependencias

### Dependencias de producción

```bash
npm install \
  react-router-dom \
  axios \
  zustand \
  react-hook-form \
  @hookform/resolvers \
  zod \
  lucide-react \
  clsx \
  tailwind-merge \
  class-variance-authority
```

| Paquete | Rol |
|---|---|
| `react-router-dom` | Routing declarativo (React Router v6) |
| `axios` | Cliente HTTP con soporte de interceptors |
| `zustand` | Gestión de estado global (consistente con RN) |
| `react-hook-form` | Formularios con validación de alto rendimiento |
| `@hookform/resolvers` | Adaptador para integrar Zod con react-hook-form |
| `zod` | Esquemas de validación tipados |
| `lucide-react` | Iconos SVG modernos |
| `clsx` + `tailwind-merge` | Combinar clases Tailwind condicionalmente |
| `class-variance-authority` | Variantes de componentes (base de shadcn) |

### Dependencias de desarrollo

```bash
npm install -D \
  tailwindcss \
  @tailwindcss/vite \
  @types/node
```

---

## 1.3 Configurar Tailwind CSS

### Actualizar `vite.config.ts`

**Archivo:** `vite.config.ts`

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'
import path from 'path'

export default defineConfig({
  plugins: [
    react(),
    tailwindcss(),
  ],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
})
```

### Actualizar `tsconfig.json`

Agrega el alias `@` para importaciones absolutas:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

> Si tu proyecto generó `tsconfig.app.json` y `tsconfig.node.json` por separado, agrega el bloque `paths` dentro de `tsconfig.app.json`.

### Crear `src/index.css`

Reemplaza el contenido de `src/index.css`:

```css
/* src/index.css */
@import "tailwindcss";

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 221.2 83.2% 53.3%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 224.3 76.3% 48%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

---

## 1.4 Instalar y configurar shadcn/ui

```bash
npx shadcn@latest init
```

Responde las preguntas del asistente:

```
Which style would you like to use? › Default
Which color would you like to use as base color? › Slate
Where is your global CSS file? › src/index.css
Would you like to use CSS variables for colors? › yes
Where is your tailwind.config located? › (enter, no aplica con Tailwind v4)
Configure the import alias for components: › @/presentation/components/ui
Configure the import alias for utils: › @/core/utils
```

Instala los componentes base que usaremos en el tutorial:

```bash
npx shadcn@latest add button input label card badge
npx shadcn@latest add form select textarea
npx shadcn@latest add dialog alert-dialog
npx shadcn@latest add table
npx shadcn@latest add dropdown-menu
npx shadcn@latest add toast
npx shadcn@latest add skeleton
npx shadcn@latest add avatar
npx shadcn@latest add separator
```

> shadcn/ui genera los componentes directamente en tu proyecto — no son una dependencia npm, son código tuyo que puedes modificar libremente.

---

## 1.5 Crear la estructura Clean Architecture

```bash
# Desde la raíz del proyecto
mkdir -p src/data/{api,storage}
mkdir -p src/domain/{model,store}
mkdir -p src/presentation/{pages/{auth,catalog,cart,orders,profile,admin},components/ui,router}
mkdir -p src/theme
mkdir -p src/core/{config,utils}
```

Estructura final esperada:

```
src/
├── data/
│   ├── api/              ← axios client, interceptors, *.api.ts
│   └── storage/          ← wrapper de localStorage
├── domain/
│   ├── model/            ← interfaces y tipos TypeScript (*.types.ts)
│   └── store/            ← stores Zustand (*.store.ts)
├── presentation/
│   ├── pages/
│   │   ├── auth/         ← LoginPage, RegisterPage
│   │   ├── catalog/      ← CatalogPage, ProductDetailPage
│   │   ├── cart/         ← CartPage
│   │   ├── orders/       ← OrdersPage, OrderDetailPage
│   │   ├── profile/      ← ProfilePage
│   │   └── admin/        ← Dashboard, Categories, Products, Orders, Users
│   ├── components/
│   │   └── ui/           ← componentes shadcn/ui generados
│   └── router/           ← AppRouter, rutas protegidas
├── theme/                ← tokens de diseño adicionales
└── core/
    ├── config/           ← api.config.ts, env vars
    └── utils/            ← cn helper, formatters
```

---

## 1.6 Archivos de configuración base

### `src/core/config/api.config.ts`

```typescript
// src/core/config/api.config.ts

export const API_CONFIG = {
  BASE_URL: import.meta.env.VITE_API_BASE_URL ?? 'http://localhost:8000/api',
  TIMEOUT: 10_000,
} as const
```

### `src/core/utils/cn.ts`

Utilidad para combinar clases Tailwind (helper estándar de shadcn):

```typescript
// src/core/utils/cn.ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

### `src/core/utils/formatters.ts`

```typescript
// src/core/utils/formatters.ts

/**
 * Formatea un número como precio en dólares.
 * Ejemplo: 1234.5 → "$1,234.50"
 */
export function formatPrice(amount: number): string {
  return new Intl.NumberFormat('en-US', {
    style: 'currency',
    currency: 'USD',
  }).format(amount)
}

/**
 * Formatea una fecha ISO a formato legible.
 * Ejemplo: "2024-03-15T10:30:00Z" → "Mar 15, 2024"
 */
export function formatDate(iso: string): string {
  return new Intl.DateTimeFormat('en-US', {
    year: 'numeric',
    month: 'short',
    day: 'numeric',
  }).format(new Date(iso))
}
```

### `src/theme/colors.ts`

```typescript
// src/theme/colors.ts

/**
 * Paleta de colores semánticos de la app.
 * Usados cuando se necesita un valor JS (canvas, charts, etc.).
 * Para clases Tailwind usar directamente: text-primary, bg-destructive, etc.
 */
export const colors = {
  primary: 'hsl(221.2 83.2% 53.3%)',
  primaryForeground: 'hsl(210 40% 98%)',
  destructive: 'hsl(0 84.2% 60.2%)',
  muted: 'hsl(210 40% 96.1%)',
  mutedForeground: 'hsl(215.4 16.3% 46.9%)',
  border: 'hsl(214.3 31.8% 91.4%)',
  background: 'hsl(0 0% 100%)',
  foreground: 'hsl(222.2 84% 4.9%)',
} as const
```

### `.env`

Crea el archivo de variables de entorno en la raíz del proyecto:

```env
VITE_API_BASE_URL=http://localhost:8000/api
```

> Vite expone solo las variables que empiecen con `VITE_`. Nunca incluyas secretos en este archivo si es un proyecto público.

---

## 1.7 Páginas placeholder

Crea una página placeholder para cada sección. Las reemplazaremos módulo a módulo.

### `src/presentation/pages/auth/LoginPage.tsx`

```tsx
// src/presentation/pages/auth/LoginPage.tsx
export default function LoginPage() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <p className="text-muted-foreground">Login — Módulo 3</p>
    </div>
  )
}
```

### `src/presentation/pages/auth/RegisterPage.tsx`

```tsx
// src/presentation/pages/auth/RegisterPage.tsx
export default function RegisterPage() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <p className="text-muted-foreground">Register — Módulo 3</p>
    </div>
  )
}
```

### `src/presentation/pages/catalog/CatalogPage.tsx`

```tsx
// src/presentation/pages/catalog/CatalogPage.tsx
export default function CatalogPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Catálogo — Módulo 4</p>
    </div>
  )
}
```

### `src/presentation/pages/catalog/ProductDetailPage.tsx`

```tsx
// src/presentation/pages/catalog/ProductDetailPage.tsx
export default function ProductDetailPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Detalle de producto — Módulo 6</p>
    </div>
  )
}
```

### `src/presentation/pages/cart/CartPage.tsx`

```tsx
// src/presentation/pages/cart/CartPage.tsx
export default function CartPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Carrito — Módulo 6</p>
    </div>
  )
}
```

### `src/presentation/pages/orders/OrdersPage.tsx`

```tsx
// src/presentation/pages/orders/OrdersPage.tsx
export default function OrdersPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Órdenes — Módulo 7</p>
    </div>
  )
}
```

### `src/presentation/pages/orders/OrderDetailPage.tsx`

```tsx
// src/presentation/pages/orders/OrderDetailPage.tsx
export default function OrderDetailPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Detalle de orden — Módulo 7</p>
    </div>
  )
}
```

### `src/presentation/pages/profile/ProfilePage.tsx`

```tsx
// src/presentation/pages/profile/ProfilePage.tsx
export default function ProfilePage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Perfil — Módulo 8</p>
    </div>
  )
}
```

### `src/presentation/pages/admin/AdminDashboardPage.tsx`

```tsx
// src/presentation/pages/admin/AdminDashboardPage.tsx
export default function AdminDashboardPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Admin Dashboard — Módulo 9</p>
    </div>
  )
}
```

### `src/presentation/pages/admin/AdminCategoriesPage.tsx`

```tsx
// src/presentation/pages/admin/AdminCategoriesPage.tsx
export default function AdminCategoriesPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Admin Categorías — Módulo 10</p>
    </div>
  )
}
```

### `src/presentation/pages/admin/AdminProductsPage.tsx`

```tsx
// src/presentation/pages/admin/AdminProductsPage.tsx
export default function AdminProductsPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Admin Productos — Módulo 11</p>
    </div>
  )
}
```

### `src/presentation/pages/admin/AdminOrdersPage.tsx`

```tsx
// src/presentation/pages/admin/AdminOrdersPage.tsx
export default function AdminOrdersPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Admin Órdenes — Módulo 12</p>
    </div>
  )
}
```

### `src/presentation/pages/admin/AdminUsersPage.tsx`

```tsx
// src/presentation/pages/admin/AdminUsersPage.tsx
export default function AdminUsersPage() {
  return (
    <div className="p-8">
      <p className="text-muted-foreground">Admin Usuarios — Módulo 13</p>
    </div>
  )
}
```

---

## 1.8 React Router — AppRouter

**Archivo:** `src/presentation/router/AppRouter.tsx`

```tsx
// src/presentation/router/AppRouter.tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom'
import { Suspense, lazy } from 'react'

// Auth
const LoginPage = lazy(() => import('../pages/auth/LoginPage'))
const RegisterPage = lazy(() => import('../pages/auth/RegisterPage'))

// Catalog
const CatalogPage = lazy(() => import('../pages/catalog/CatalogPage'))
const ProductDetailPage = lazy(() => import('../pages/catalog/ProductDetailPage'))

// Cart
const CartPage = lazy(() => import('../pages/cart/CartPage'))

// Orders
const OrdersPage = lazy(() => import('../pages/orders/OrdersPage'))
const OrderDetailPage = lazy(() => import('../pages/orders/OrderDetailPage'))

// Profile
const ProfilePage = lazy(() => import('../pages/profile/ProfilePage'))

// Admin
const AdminDashboardPage = lazy(() => import('../pages/admin/AdminDashboardPage'))
const AdminCategoriesPage = lazy(() => import('../pages/admin/AdminCategoriesPage'))
const AdminProductsPage = lazy(() => import('../pages/admin/AdminProductsPage'))
const AdminOrdersPage = lazy(() => import('../pages/admin/AdminOrdersPage'))
const AdminUsersPage = lazy(() => import('../pages/admin/AdminUsersPage'))

function PageLoader() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="h-8 w-8 animate-spin rounded-full border-4 border-primary border-t-transparent" />
    </div>
  )
}

export default function AppRouter() {
  return (
    <BrowserRouter>
      <Suspense fallback={<PageLoader />}>
        <Routes>
          {/* Auth */}
          <Route path="/login" element={<LoginPage />} />
          <Route path="/register" element={<RegisterPage />} />

          {/* Catalog (público) */}
          <Route path="/" element={<CatalogPage />} />
          <Route path="/catalog" element={<CatalogPage />} />
          <Route path="/products/:id" element={<ProductDetailPage />} />

          {/* Protegidas (se implementan en módulo 3) */}
          <Route path="/cart" element={<CartPage />} />
          <Route path="/orders" element={<OrdersPage />} />
          <Route path="/orders/:id" element={<OrderDetailPage />} />
          <Route path="/profile" element={<ProfilePage />} />

          {/* Admin */}
          <Route path="/admin" element={<AdminDashboardPage />} />
          <Route path="/admin/categories" element={<AdminCategoriesPage />} />
          <Route path="/admin/products" element={<AdminProductsPage />} />
          <Route path="/admin/orders" element={<AdminOrdersPage />} />
          <Route path="/admin/users" element={<AdminUsersPage />} />

          {/* Fallback */}
          <Route path="*" element={<Navigate to="/" replace />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  )
}
```

---

## 1.9 Componente raíz y punto de entrada

### Limpiar `src/App.tsx`

```tsx
// src/App.tsx
import AppRouter from './presentation/router/AppRouter'

export default function App() {
  return <AppRouter />
}
```

### Actualizar `src/main.tsx`

```tsx
// src/main.tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import App from './App'

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>,
)
```

### Eliminar archivos innecesarios

```bash
rm src/App.css
rm -rf src/assets/react.svg
```

---

## 1.10 Verificar el setup

Arranca la app:

```bash
npm run dev
```

Verifica en el navegador:
- `http://localhost:5173` → muestra "Catálogo — Módulo 4"
- `http://localhost:5173/login` → muestra "Login — Módulo 3"
- `http://localhost:5173/admin` → muestra "Admin Dashboard — Módulo 9"

Verifica que no hay errores de TypeScript:

```bash
npx tsc --noEmit
```

---

## 1.11 Estructura de archivos del módulo

```
src/
├── core/
│   ├── config/
│   │   └── api.config.ts         ← URL del backend y timeout
│   └── utils/
│       ├── cn.ts                 ← helper tailwind-merge
│       └── formatters.ts         ← formatPrice, formatDate
├── theme/
│   └── colors.ts                 ← tokens de color (valores JS)
├── presentation/
│   ├── pages/
│   │   ├── auth/
│   │   │   ├── LoginPage.tsx     ← placeholder (módulo 3)
│   │   │   └── RegisterPage.tsx  ← placeholder (módulo 3)
│   │   ├── catalog/
│   │   │   ├── CatalogPage.tsx         ← placeholder (módulo 4)
│   │   │   └── ProductDetailPage.tsx   ← placeholder (módulo 6)
│   │   ├── cart/
│   │   │   └── CartPage.tsx      ← placeholder (módulo 6)
│   │   ├── orders/
│   │   │   ├── OrdersPage.tsx    ← placeholder (módulo 7)
│   │   │   └── OrderDetailPage.tsx
│   │   ├── profile/
│   │   │   └── ProfilePage.tsx   ← placeholder (módulo 8)
│   │   └── admin/
│   │       ├── AdminDashboardPage.tsx
│   │       ├── AdminCategoriesPage.tsx
│   │       ├── AdminProductsPage.tsx
│   │       ├── AdminOrdersPage.tsx
│   │       └── AdminUsersPage.tsx
│   ├── components/
│   │   └── ui/                   ← componentes shadcn/ui
│   └── router/
│       └── AppRouter.tsx         ← React Router v6 con lazy loading
├── App.tsx
├── main.tsx
└── index.css
```

---

## Resumen

| Concepto | Detalle |
|---|---|
| Bundler | Vite 5 + `@vitejs/plugin-react` |
| Lenguaje | TypeScript 5 con alias `@/` |
| Estilos | Tailwind CSS v4 + variables CSS + shadcn/ui |
| Routing | React Router v6 con `lazy` + `Suspense` |
| Estructura | Clean Architecture global: `data/`, `domain/`, `presentation/` |
| Configuración | `VITE_API_BASE_URL` en `.env` |
| Utils | `cn()` (tailwind-merge) + `formatPrice` + `formatDate` |
