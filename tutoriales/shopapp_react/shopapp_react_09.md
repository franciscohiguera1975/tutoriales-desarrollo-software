# ShopApp React Web — Módulo 9

## Admin Dashboard — Shell de administración y KPIs

**Duración estimada:** 2–3 horas
**Prerequisitos:** Módulos 1–8 completados. El usuario autenticado debe tener `is_staff: true` para
acceder a las rutas de administración.

---

> **Objetivo**
> Construir el panel de administración de ShopApp: un shell con sidebar de navegación responsivo,
> un store Zustand que agrega estadísticas obtenidas en paralelo desde múltiples endpoints,
> y una página de dashboard con tarjetas de KPIs que muestran métricas clave del negocio.
> Las rutas `/admin/*` estarán protegidas para usuarios staff únicamente.
>
> **Checkpoint final**
> - La ruta `/admin` solo es accesible para usuarios con `is_staff: true`; los demás son
>   redirigidos.
> - `AdminShell` renderiza un sidebar de navegación en escritorio y un menú tipo Sheet en móvil.
> - `AdminDashboardPage` muestra 6 `KpiCard` con datos reales del backend.
> - Los KPIs de "Órdenes Pendientes" y "Sin Stock" tienen variante visual de alerta.
> - Durante la carga se muestran 6 skeleton cards.

---

## 9.1 Tipos de administración

**Archivo:** `src/domain/model/admin.types.ts`

Este archivo define `AdminStats`, el objeto que agrupa todas las métricas del dashboard. Dado que
el backend no expone un endpoint específico de estadísticas, los valores se calculan en la capa de
datos combinando respuestas paginadas de varios endpoints; aquí solo definimos el contrato de
salida. Centralizar los tipos en esta capa permite que tanto el store como los componentes dependan
de una interfaz estable, independiente de cómo se obtengan los datos.

```typescript
// src/domain/model/admin.types.ts

export interface AdminStats {
  /** Número total de productos registrados en el sistema. */
  total_products: number;

  /** Número total de categorías de productos. */
  total_categories: number;

  /** Número total de órdenes realizadas. */
  total_orders: number;

  /** Número total de usuarios registrados. */
  total_users: number;

  /** Órdenes en estado pendiente de procesamiento. */
  pending_orders: number;

  /** Productos con stock igual a cero. */
  out_of_stock_products: number;
}
```

---

## 9.2 API de administración

**Archivo:** `src/data/api/admin.api.ts`

Este módulo implementa `getAdminStats()`, que lanza cuatro solicitudes en paralelo usando
`Promise.all` para minimizar la latencia total. Cada endpoint paginado de Django REST Framework
devuelve un campo `count` con el total de registros, que usamos para poblar `AdminStats`.
Los endpoints de productos y órdenes reciben `page_size=1` para que el backend devuelva el mínimo
de datos (solo necesitamos el campo `count`).

```typescript
// src/data/api/admin.api.ts

import { apiClient } from './api.client';
import type { AdminStats } from '../../domain/model/admin.types';

/** Respuesta estándar paginada de Django REST Framework. */
interface PaginatedResponse<T = unknown> {
  count: number;
  next: string | null;
  previous: string | null;
  results: T[];
}

/** Estructura mínima de un producto, solo los campos que usamos para las stats. */
interface ProductSummary {
  id: number;
  stock: number;
}

/**
 * Obtiene las estadísticas del panel de administración.
 *
 * Realiza cuatro solicitudes en paralelo para minimizar el tiempo de espera:
 *  - GET /products/?page_size=100  → total_products + out_of_stock_products
 *  - GET /categories/              → total_categories
 *  - GET /orders/?page_size=1      → total_orders
 *  - GET /users/?page_size=1       → total_users
 *
 * Para pending_orders se filtra adicionalmente con status=pending.
 */
export async function getAdminStats(): Promise<AdminStats> {
  const [productsRes, categoriesRes, ordersRes, usersRes, pendingOrdersRes] =
    await Promise.all([
      // Pedimos hasta 100 productos para poder contar los sin stock.
      // En producción real, el backend debería proveer este dato directamente.
      apiClient.get<PaginatedResponse<ProductSummary>>(
        '/products/?page_size=100'
      ),
      apiClient.get<PaginatedResponse>('/categories/'),
      apiClient.get<PaginatedResponse>('/orders/?page_size=1'),
      apiClient.get<PaginatedResponse>('/users/?page_size=1'),
      apiClient.get<PaginatedResponse>('/orders/?page_size=1&status=pending'),
    ]);

  // Contamos los productos con stock = 0 a partir de los resultados recibidos
  const outOfStock = productsRes.data.results.filter(
    (p) => p.stock === 0
  ).length;

  return {
    total_products: productsRes.data.count,
    total_categories: categoriesRes.data.count,
    total_orders: ordersRes.data.count,
    total_users: usersRes.data.count,
    pending_orders: pendingOrdersRes.data.count,
    out_of_stock_products: outOfStock,
  };
}
```

> **Nota de diseño:** La consulta de productos con `page_size=100` es una solución pragmática para
> un backend sin un endpoint de stats. En un sistema con muchos productos, lo correcto es añadir al
> backend un endpoint `GET /admin/stats/` que devuelva `AdminStats` directamente con una sola
> consulta a la base de datos. En Módulos posteriores se muestra cómo construirlo en Django.

---

## 9.3 AdminStore

**Archivo:** `src/domain/store/admin.store.ts`

El store de administración sigue el mismo patrón de los stores anteriores: estado reactivo con
Zustand y una acción asíncrona `fetchStats` que llama a la capa de datos. Se mantiene separado
del `profileStore` y `authStore` para respetar el principio de responsabilidad única. Varios
componentes del panel pueden suscribirse a este store sin acoplarse entre sí.

```typescript
// src/domain/store/admin.store.ts

import { create } from 'zustand';
import type { AdminStats } from '../model/admin.types';
import { getAdminStats } from '../../data/api/admin.api';

interface AdminState {
  stats: AdminStats | null;
  isLoading: boolean;
  error: string | null;

  fetchStats: () => Promise<void>;
  resetStats: () => void;
}

export const useAdminStore = create<AdminState>((set) => ({
  stats: null,
  isLoading: false,
  error: null,

  fetchStats: async () => {
    set({ isLoading: true, error: null });
    try {
      const stats = await getAdminStats();
      set({ stats, isLoading: false });
    } catch (err) {
      const message =
        err instanceof Error
          ? err.message
          : 'No se pudieron cargar las estadísticas.';
      set({ error: message, isLoading: false });
    }
  },

  resetStats: () => set({ stats: null, error: null }),
}));
```

---

## 9.4 Componente AdminShell

**Archivo:** `src/presentation/components/AdminShell.tsx`

`AdminShell` es el layout que envuelve todas las páginas del panel de administración. En pantallas
grandes muestra un sidebar fijo con los enlaces de navegación; en móvil, el sidebar se reemplaza
por un botón de menú hamburguesa que abre un `Sheet` (cajón lateral) de shadcn/ui. Cada enlace
se resalta visualmente cuando la ruta actual coincide usando `useLocation` de React Router. El
componente `NavLink` se encapsula en una función interna para mantener el código DRY.

Instala los componentes necesarios si aún no los tienes:

```bash
npx shadcn@latest add sheet separator
```

```tsx
// src/presentation/components/AdminShell.tsx

import { useState } from 'react';
import { Link, useLocation } from 'react-router-dom';
import {
  LayoutDashboard,
  Tag,
  Package,
  ShoppingCart,
  Users,
  Menu,
  ArrowLeft,
} from 'lucide-react';

import { Button } from '@/components/ui/button';
import { Separator } from '@/components/ui/separator';
import {
  Sheet,
  SheetContent,
  SheetHeader,
  SheetTitle,
  SheetTrigger,
} from '@/components/ui/sheet';
import { cn } from '@/lib/utils';

// ── Definición de los enlaces de navegación ──────────────────────────────────

interface NavItem {
  label: string;
  href: string;
  icon: React.ElementType;
}

const navItems: NavItem[] = [
  { label: 'Dashboard', href: '/admin', icon: LayoutDashboard },
  { label: 'Categorías', href: '/admin/categories', icon: Tag },
  { label: 'Productos', href: '/admin/products', icon: Package },
  { label: 'Órdenes', href: '/admin/orders', icon: ShoppingCart },
  { label: 'Usuarios', href: '/admin/users', icon: Users },
];

// ── Subcomponente: un enlace de la barra lateral ─────────────────────────────

interface SideNavLinkProps {
  item: NavItem;
  currentPath: string;
  onClick?: () => void;
}

function SideNavLink({ item, currentPath, onClick }: SideNavLinkProps) {
  // El enlace de Dashboard solo se activa en la ruta exacta
  const isActive =
    item.href === '/admin'
      ? currentPath === '/admin'
      : currentPath.startsWith(item.href);

  const Icon = item.icon;

  return (
    <Link
      to={item.href}
      onClick={onClick}
      className={cn(
        'flex items-center gap-3 rounded-md px-3 py-2 text-sm font-medium transition-colors',
        isActive
          ? 'bg-primary/10 text-primary'
          : 'text-muted-foreground hover:bg-muted hover:text-foreground'
      )}
    >
      <Icon className="h-4 w-4 shrink-0" />
      {item.label}
    </Link>
  );
}

// ── Subcomponente: contenido del sidebar (reutilizado en desktop y Sheet) ─────

interface SidebarContentProps {
  currentPath: string;
  onLinkClick?: () => void;
}

function SidebarContent({ currentPath, onLinkClick }: SidebarContentProps) {
  return (
    <div className="flex h-full flex-col gap-2">
      {/* Título del panel */}
      <div className="px-3 py-4">
        <h2 className="text-xs font-semibold uppercase tracking-widest text-muted-foreground">
          Panel Admin
        </h2>
      </div>

      {/* Navegación principal */}
      <nav className="flex-1 space-y-1 px-2">
        {navItems.map((item) => (
          <SideNavLink
            key={item.href}
            item={item}
            currentPath={currentPath}
            onClick={onLinkClick}
          />
        ))}
      </nav>

      {/* Enlace de regreso a la tienda */}
      <div className="px-2 pb-4">
        <Separator className="mb-4" />
        <Link
          to="/"
          onClick={onLinkClick}
          className="flex items-center gap-3 rounded-md px-3 py-2 text-sm font-medium text-muted-foreground hover:bg-muted hover:text-foreground transition-colors"
        >
          <ArrowLeft className="h-4 w-4 shrink-0" />
          Volver a la tienda
        </Link>
      </div>
    </div>
  );
}

// ── Componente principal AdminShell ──────────────────────────────────────────

interface AdminShellProps {
  children: React.ReactNode;
}

export function AdminShell({ children }: AdminShellProps) {
  const { pathname } = useLocation();
  const [sheetOpen, setSheetOpen] = useState(false);

  return (
    <div className="flex min-h-screen bg-background">
      {/* ── Sidebar de escritorio ─────────────────────────────────────── */}
      <aside className="hidden md:flex w-56 shrink-0 flex-col border-r bg-card">
        <SidebarContent currentPath={pathname} />
      </aside>

      {/* ── Contenido principal ───────────────────────────────────────── */}
      <div className="flex flex-1 flex-col">
        {/* Barra superior móvil */}
        <header className="flex h-14 items-center gap-4 border-b px-4 md:hidden">
          <Sheet open={sheetOpen} onOpenChange={setSheetOpen}>
            <SheetTrigger asChild>
              <Button variant="ghost" size="icon" aria-label="Abrir menú">
                <Menu className="h-5 w-5" />
              </Button>
            </SheetTrigger>
            <SheetContent side="left" className="w-56 p-0">
              <SheetHeader className="sr-only">
                <SheetTitle>Navegación de administración</SheetTitle>
              </SheetHeader>
              <SidebarContent
                currentPath={pathname}
                onLinkClick={() => setSheetOpen(false)}
              />
            </SheetContent>
          </Sheet>
          <span className="font-semibold text-sm">Panel Admin</span>
        </header>

        {/* Área de contenido de la página */}
        <main className="flex-1 overflow-auto p-6">{children}</main>
      </div>
    </div>
  );
}
```

---

## 9.5 Componente KpiCard

**Archivo:** `src/presentation/components/KpiCard.tsx`

`KpiCard` es un componente de presentación puro (no conectado a ningún store) que recibe sus
datos como props. El `variant` controla el color del borde de la tarjeta: `warning` usa un borde
ámbar para llamar la atención sobre métricas que requieren seguimiento, y `danger` usa rojo para
indicar una situación crítica que necesita acción inmediata. El icono se renderiza con un fondo
suave del color del borde para cohesión visual.

```tsx
// src/presentation/components/KpiCard.tsx

import type { LucideIcon } from 'lucide-react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { cn } from '@/lib/utils';

export interface KpiCardProps {
  title: string;
  value: number | string;
  icon: LucideIcon;
  description?: string;
  variant?: 'default' | 'warning' | 'danger';
}

const variantStyles = {
  default: {
    card: 'border-border',
    iconWrapper: 'bg-primary/10 text-primary',
  },
  warning: {
    card: 'border-amber-400 dark:border-amber-500',
    iconWrapper: 'bg-amber-100 text-amber-600 dark:bg-amber-900/30 dark:text-amber-400',
  },
  danger: {
    card: 'border-destructive',
    iconWrapper: 'bg-destructive/10 text-destructive',
  },
};

export function KpiCard({
  title,
  value,
  icon: Icon,
  description,
  variant = 'default',
}: KpiCardProps) {
  const styles = variantStyles[variant];

  return (
    <Card className={cn('border-2 transition-shadow hover:shadow-md', styles.card)}>
      <CardHeader className="flex flex-row items-center justify-between pb-2 space-y-0">
        <CardTitle className="text-sm font-medium text-muted-foreground">
          {title}
        </CardTitle>
        <div className={cn('rounded-md p-2', styles.iconWrapper)}>
          <Icon className="h-4 w-4" />
        </div>
      </CardHeader>
      <CardContent>
        <div className="text-3xl font-bold tracking-tight">
          {typeof value === 'number' ? value.toLocaleString('es-EC') : value}
        </div>
        {description && (
          <p className="text-xs text-muted-foreground mt-1">{description}</p>
        )}
      </CardContent>
    </Card>
  );
}
```

---

## 9.6 AdminDashboardPage

**Archivo:** `src/presentation/pages/admin/AdminDashboardPage.tsx`

La página principal del panel obtiene las estadísticas al montarse mediante `fetchStats` del
`useAdminStore`. Mientras los datos cargan, muestra 6 esqueletos de tarjetas usando el componente
`Skeleton` de shadcn para evitar el efecto de "flash" de contenido vacío. Si la carga tiene éxito,
renderiza una cuadrícula responsiva de 6 `KpiCard`, asignando las variantes `warning` y `danger`
a los KPIs que representan situaciones que requieren atención.

Instala el componente Skeleton si aún no lo tienes:

```bash
npx shadcn@latest add skeleton
```

```tsx
// src/presentation/pages/admin/AdminDashboardPage.tsx

import { useEffect } from 'react';
import {
  Package,
  Tag,
  ShoppingCart,
  Users,
  Clock,
  AlertTriangle,
} from 'lucide-react';

import { AdminShell } from '../../components/AdminShell';
import { KpiCard } from '../../components/KpiCard';
import { useAdminStore } from '../../../domain/store/admin.store';
import { Skeleton } from '@/components/ui/skeleton';
import { Card, CardContent, CardHeader } from '@/components/ui/card';

// ── Skeleton de una KpiCard (para el estado de carga) ────────────────────────

function KpiCardSkeleton() {
  return (
    <Card className="border-2">
      <CardHeader className="flex flex-row items-center justify-between pb-2 space-y-0">
        <Skeleton className="h-4 w-28" />
        <Skeleton className="h-8 w-8 rounded-md" />
      </CardHeader>
      <CardContent>
        <Skeleton className="h-8 w-16 mb-1" />
        <Skeleton className="h-3 w-32" />
      </CardContent>
    </Card>
  );
}

// ── Componente principal ─────────────────────────────────────────────────────

export default function AdminDashboardPage() {
  const { stats, isLoading, error, fetchStats } = useAdminStore();

  useEffect(() => {
    fetchStats();
  }, [fetchStats]);

  return (
    <AdminShell>
      <div className="space-y-6">
        {/* Encabezado de la página */}
        <div>
          <h1 className="text-2xl font-bold tracking-tight">Dashboard</h1>
          <p className="text-muted-foreground text-sm mt-1">
            Resumen general del estado de la tienda.
          </p>
        </div>

        {/* Estado de error */}
        {error && !isLoading && (
          <div className="rounded-md bg-destructive/10 border border-destructive/30 px-4 py-3">
            <p className="text-sm text-destructive font-medium">{error}</p>
          </div>
        )}

        {/* Cuadrícula de KPIs */}
        <div className="grid gap-4 grid-cols-1 sm:grid-cols-2 lg:grid-cols-3">
          {isLoading ? (
            // Skeleton cards mientras carga
            Array.from({ length: 6 }).map((_, i) => (
              <KpiCardSkeleton key={i} />
            ))
          ) : (
            <>
              <KpiCard
                title="Total Productos"
                value={stats?.total_products ?? 0}
                icon={Package}
                description="Productos registrados en el catálogo"
              />
              <KpiCard
                title="Categorías"
                value={stats?.total_categories ?? 0}
                icon={Tag}
                description="Categorías de productos activas"
              />
              <KpiCard
                title="Total Órdenes"
                value={stats?.total_orders ?? 0}
                icon={ShoppingCart}
                description="Pedidos realizados hasta la fecha"
              />
              <KpiCard
                title="Total Usuarios"
                value={stats?.total_users ?? 0}
                icon={Users}
                description="Cuentas registradas en la plataforma"
              />
              <KpiCard
                title="Órdenes Pendientes"
                value={stats?.pending_orders ?? 0}
                icon={Clock}
                description="Pedidos sin confirmar ni procesar"
                variant="warning"
              />
              <KpiCard
                title="Sin Stock"
                value={stats?.out_of_stock_products ?? 0}
                icon={AlertTriangle}
                description="Productos con stock igual a cero"
                variant="danger"
              />
            </>
          )}
        </div>
      </div>
    </AdminShell>
  );
}
```

---

## 9.7 Actualizar AppRouter con rutas de administración

**Archivo:** `src/presentation/router/AppRouter.tsx` (modificación)

En este paso añadimos un `ProtectedRoute` más estricto que verifica si el usuario tiene
`is_staff: true`. Si un usuario autenticado pero sin permisos de staff intenta acceder a
`/admin/*`, es redirigido al catálogo. Si no está autenticado, es redirigido al login.

### Crear el componente StaffRoute

Primero añade un componente que extiende `ProtectedRoute` con la verificación de staff. Si ya
tienes `ProtectedRoute` en tu proyecto, puedes añadir `StaffRoute` en el mismo archivo:

```tsx
// src/presentation/router/ProtectedRoute.tsx
// ── Añadir al final del archivo existente ────────────────────────────────────

import { Navigate } from 'react-router-dom';
import { useAuthStore } from '../../domain/store/auth.store';

/**
 * Ruta protegida para usuarios con is_staff: true.
 * Si el usuario no está autenticado → redirige a /auth/login.
 * Si está autenticado pero no es staff → redirige a /catalog.
 */
export function StaffRoute({ children }: { children: React.ReactNode }) {
  const { user } = useAuthStore();

  if (!user) {
    return <Navigate to="/auth/login" replace />;
  }

  if (!user.is_staff) {
    return <Navigate to="/catalog" replace />;
  }

  return <>{children}</>;
}
```

### Registrar las rutas de administración

```tsx
// src/presentation/router/AppRouter.tsx
// ── Añadir los imports necesarios ────────────────────────────────────────────

import { StaffRoute } from './ProtectedRoute';
import AdminDashboardPage from '../pages/admin/AdminDashboardPage';

// Las páginas de secciones de admin se crearán en módulos posteriores;
// por ahora registramos solo el dashboard principal.

// ── Dentro del árbol de rutas ────────────────────────────────────────────────

<Route
  path="/admin"
  element={
    <StaffRoute>
      <AdminDashboardPage />
    </StaffRoute>
  }
/>

{/*
  Las rutas siguientes se añadirán en los Módulos 10-12:

  <Route
    path="/admin/categories"
    element={<StaffRoute><AdminCategoriesPage /></StaffRoute>}
  />
  <Route
    path="/admin/products"
    element={<StaffRoute><AdminProductsPage /></StaffRoute>}
  />
  <Route
    path="/admin/orders"
    element={<StaffRoute><AdminOrdersPage /></StaffRoute>}
  />
  <Route
    path="/admin/users"
    element={<StaffRoute><AdminUsersPage /></StaffRoute>}
  />
*/}
```

> **Sobre el `AdminShell` y el layout:** en este módulo `AdminShell` es invocado directamente
> dentro de cada página de administración (como se ve en `AdminDashboardPage`). Esto permite que
> cada página pueda añadir su propio encabezado dentro del shell sin configuración adicional en
> el router. Si prefieres un layout centralizado en el router, puedes usar un `<Route>` padre con
> `element={<StaffRoute><AdminShell /></StaffRoute>}` y rutas hijas con `<Outlet />`, pero deberás
> extraer el `<main>` del `AdminShell` hacia un `<Outlet />`.

---

## 9.8 Checkpoint y pruebas manuales

Antes de continuar al Módulo 10, verifica que todo el módulo de administración funciona:

1. **Acceso con usuario regular**
   - Inicia sesión con un usuario que tenga `is_staff: false`.
   - Intenta navegar a `/admin` directamente desde la barra de direcciones.
   - Debes ser redirigido a `/catalog` automáticamente.

2. **Acceso con usuario staff**
   - Inicia sesión con un usuario que tenga `is_staff: true`.
   - Navega a `/admin`.
   - Deben cargarse las 6 KpiCards con datos reales del backend.

3. **Skeleton de carga**
   - Abre DevTools > Network y activa "Slow 3G" (o usa throttling).
   - Recarga `/admin` y verifica que aparecen los 6 skeleton cards antes de los datos reales.

4. **Sidebar de escritorio**
   - En una ventana de más de 768px de ancho, verifica que el sidebar izquierdo está visible con
     los 5 enlaces de navegación.
   - El enlace "Dashboard" debe aparecer resaltado con `bg-primary/10 text-primary`.

5. **Menú móvil (Sheet)**
   - Reduce el ancho del navegador a menos de 768px.
   - Verifica que el sidebar se oculta y aparece el botón hamburguesa en la barra superior.
   - Al hacer clic en el botón, se abre el Sheet lateral con la misma navegación.
   - Al hacer clic en un enlace dentro del Sheet, éste se cierra automáticamente.

6. **Variantes de KpiCard**
   - Las tarjetas "Órdenes Pendientes" y "Sin Stock" deben tener bordes de color diferente al
     resto (ámbar y rojo respectivamente).

7. **Enlace "Volver a la tienda"**
   - Al hacer clic en "Volver a la tienda" en el sidebar, la navegación te lleva a `/`.
   - La ruta actual es `/`, por lo que el sidebar del admin ya no está visible.

8. **Link en el header para staff**
   - Desde el dropdown del header (integrado en el Módulo 8), verifica que el enlace
     "Panel de administración" solo aparece cuando `user.is_staff === true`.

---

## Resumen del módulo

| Archivo | Capa | Descripción |
|---|---|---|
| `src/domain/model/admin.types.ts` | Domain / Model | Tipo `AdminStats` con las 6 métricas del dashboard |
| `src/data/api/admin.api.ts` | Data / API | `getAdminStats()`: 5 llamadas en paralelo con `Promise.all` |
| `src/domain/store/admin.store.ts` | Domain / Store | Store Zustand con `fetchStats` y estado de carga |
| `src/presentation/components/AdminShell.tsx` | Presentation / Components | Layout con sidebar fijo (desktop) y Sheet hamburguesa (móvil) |
| `src/presentation/components/KpiCard.tsx` | Presentation / Components | Tarjeta de KPI con 3 variantes visuales y soporte para `LucideIcon` |
| `src/presentation/pages/admin/AdminDashboardPage.tsx` | Presentation / Pages | Dashboard con 6 KPIs y skeleton de carga |
| `src/presentation/router/ProtectedRoute.tsx` | Presentation / Router | Nuevo `StaffRoute` que verifica `is_staff` antes de renderizar |
| `src/presentation/router/AppRouter.tsx` | Presentation / Router | Ruta `/admin` protegida con `StaffRoute` |

### Dependencias de shadcn usadas en este módulo

| Componente | Comando de instalación |
|---|---|
| `Sheet` | `npx shadcn@latest add sheet` |
| `Separator` | `npx shadcn@latest add separator` |
| `Card` | `npx shadcn@latest add card` |
| `Button` | `npx shadcn@latest add button` |
| `Skeleton` | `npx shadcn@latest add skeleton` |

### Dependencias de lucide-react usadas en este módulo

| Icono | Uso |
|---|---|
| `LayoutDashboard` | Enlace Dashboard en el sidebar |
| `Tag` | Enlace Categorías en el sidebar |
| `Package` | Enlace Productos / KPI Total Productos |
| `ShoppingCart` | Enlace Órdenes / KPI Total Órdenes |
| `Users` | Enlace Usuarios / KPI Total Usuarios |
| `Clock` | KPI Órdenes Pendientes |
| `AlertTriangle` | KPI Sin Stock |
| `Menu` | Botón hamburguesa en móvil |
| `ArrowLeft` | Enlace "Volver a la tienda" |
