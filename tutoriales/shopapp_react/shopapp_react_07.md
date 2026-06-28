# ShopApp React Web — Módulo 7

## Órdenes — OrdersStore, Checkout y listado de órdenes

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulos 1–6 completados.

---

> **Objetivo**
> Implementar el flujo de órdenes de extremo a extremo: tipos de dominio, capa de API con `apiClient`,
> store Zustand para órdenes, página de checkout que convierte el carrito en una orden, listado
> paginado de órdenes del usuario, página de detalle de una orden y un componente `StatusBadge`
> reutilizable con colores semánticos por estado.
>
> **Checkpoint final**
> - `CheckoutPage` crea una orden con los ítems del carrito y vacía el store del carrito.
> - Al confirmar el pedido, la app navega a `/orders/:id` mostrando la orden recién creada.
> - `OrdersPage` lista todas las órdenes del usuario con badge de estado y fecha formateada.
> - `OrderDetailPage` muestra la tabla de ítems y el total de la orden.
> - `StatusBadge` aplica colores correctos para cada valor de `OrderStatus`.

---

## 7.1 Tipos del dominio de órdenes

**Archivo:** `src/domain/model/orders.types.ts`

```typescript
// src/domain/model/orders.types.ts

/**
 * Estados posibles de una orden en el backend Django.
 * Deben coincidir exactamente con los valores del campo `status` del modelo.
 */
export type OrderStatus =
  | 'pending'
  | 'confirmed'
  | 'shipped'
  | 'delivered'
  | 'cancelled'

/**
 * Un ítem dentro de una orden ya creada.
 * Guarda el id del producto (no el objeto completo) para facilitar la
 * serialización desde el backend.
 */
export interface OrderItem {
  id: number
  product: number
  product_name: string
  quantity: number
  unit_price: number
  subtotal: number
}

/**
 * Una orden completa tal y como la devuelve el backend.
 */
export interface Order {
  id: number
  user: number
  status: OrderStatus
  created_at: string   // ISO 8601
  updated_at: string   // ISO 8601
  items: OrderItem[]
  total: number
  total_items: number
}

/**
 * Payload necesario para crear una nueva orden.
 * El backend calcula precios y totales a partir de los ids de producto.
 */
export interface CreateOrderPayload {
  items: {
    product: number
    quantity: number
  }[]
}
```

Los campos `unit_price`, `subtotal` y `total` se almacenan como `number` en el frontend aunque el
backend los exponga como strings decimales. La conversión se realiza en la capa de presentación
con `formatPrice`. Si el backend usa `Decimal` serializado como string, aplica `Number(item.total)`
antes de usarlo.

---

## 7.2 Orders API

**Archivo:** `src/data/api/orders.api.ts`

```typescript
// src/data/api/orders.api.ts

import { apiClient } from './api.client'
import type { Order, CreateOrderPayload } from '@/domain/model/orders.types'

/**
 * Obtiene todas las órdenes del usuario autenticado.
 * El backend filtra automáticamente por el token JWT del interceptor.
 */
export async function getOrders(): Promise<Order[]> {
  const { data } = await apiClient.get<Order[]>('/orders/')
  return data
}

/**
 * Obtiene el detalle de una orden por su id.
 */
export async function getOrderById(id: number): Promise<Order> {
  const { data } = await apiClient.get<Order>(`/orders/${id}/`)
  return data
}

/**
 * Crea una nueva orden con los ítems del carrito.
 * Requiere autenticación (el interceptor añade el header Authorization).
 */
export async function createOrder(
  payload: CreateOrderPayload,
): Promise<Order> {
  const { data } = await apiClient.post<Order>('/orders/', payload)
  return data
}
```

Las tres funciones delegan completamente en `apiClient`, el cliente Axios con interceptores JWT
definido en `src/data/api/api.client.ts`. Si el token está caducado, el interceptor de respuesta
redirige a `/login` automáticamente, sin necesidad de manejar 401 aquí.

---

## 7.3 OrdersStore

**Archivo:** `src/domain/store/orders.store.ts`

```typescript
// src/domain/store/orders.store.ts

import { create } from 'zustand'
import { getOrders, getOrderById, createOrder } from '@/data/api/orders.api'
import { useCartStore } from './cart.store'
import type { Order } from '@/domain/model/orders.types'
import type { CartItem } from '@/domain/model/cart.types'

interface OrdersStore {
  // ── Estado ──────────────────────────────────────────────────────────────
  orders: Order[]
  currentOrder: Order | null
  isLoading: boolean
  error: string | null

  // ── Acciones ─────────────────────────────────────────────────────────────
  fetchOrders: () => Promise<void>
  fetchOrderById: (id: number) => Promise<void>
  placeOrder: (cartItems: CartItem[]) => Promise<Order>
  clearError: () => void
}

export const useOrdersStore = create<OrdersStore>((set) => ({
  // ── Estado inicial ────────────────────────────────────────────────────────
  orders: [],
  currentOrder: null,
  isLoading: false,
  error: null,

  // ── Acciones ──────────────────────────────────────────────────────────────

  /**
   * Carga todas las órdenes del usuario autenticado desde el backend.
   * Las ordena por fecha descendente (más reciente primero) en el cliente.
   */
  fetchOrders: async () => {
    set({ isLoading: true, error: null })
    try {
      const orders = await getOrders()
      const sorted = [...orders].sort(
        (a, b) =>
          new Date(b.created_at).getTime() - new Date(a.created_at).getTime(),
      )
      set({ orders: sorted })
    } catch {
      set({ error: 'No se pudieron cargar las órdenes.' })
    } finally {
      set({ isLoading: false })
    }
  },

  /**
   * Carga el detalle de una orden específica.
   */
  fetchOrderById: async (id) => {
    set({ isLoading: true, error: null, currentOrder: null })
    try {
      const order = await getOrderById(id)
      set({ currentOrder: order })
    } catch {
      set({ error: `No se pudo cargar la orden #${id}.` })
    } finally {
      set({ isLoading: false })
    }
  },

  /**
   * Crea una orden a partir de los ítems del carrito y luego vacía el carrito.
   * Devuelve la orden creada para que el componente pueda navegar a su detalle.
   *
   * @throws Relanza el error para que el componente pueda mostrarlo.
   */
  placeOrder: async (cartItems) => {
    set({ isLoading: true, error: null })
    try {
      const payload = {
        items: cartItems.map((item) => ({
          product: item.product.id,
          quantity: item.quantity,
        })),
      }

      const newOrder = await createOrder(payload)

      // Actualiza la lista de órdenes en el store
      set((state) => ({
        orders: [newOrder, ...state.orders],
        currentOrder: newOrder,
      }))

      // Vacía el carrito una vez que la orden se confirmó en el backend
      useCartStore.getState().clearCart()

      return newOrder
    } catch (err) {
      set({ error: 'No se pudo crear la orden. Inténtalo de nuevo.' })
      throw err
    } finally {
      set({ isLoading: false })
    }
  },

  clearError: () => set({ error: null }),
}))
```

`placeOrder` importa directamente el store del carrito con `useCartStore.getState()`. Esta llamada
fuera de un componente React es válida en Zustand y evita acoplamiento de props. El error se relanza
para que `CheckoutPage` pueda capturarlo y mostrarlo.

---

## 7.4 CheckoutPage

**Archivo:** `src/presentation/pages/orders/CheckoutPage.tsx`

```tsx
// src/presentation/pages/orders/CheckoutPage.tsx

import { useEffect } from 'react'
import { useNavigate, Link } from 'react-router-dom'
import { AlertCircle, Loader2, ShoppingBag } from 'lucide-react'

import { Button } from '@/components/ui/button'
import { Separator } from '@/components/ui/separator'
import { Alert, AlertDescription } from '@/components/ui/alert'

import { useCartStore } from '@/domain/store/cart.store'
import { useOrdersStore } from '@/domain/store/orders.store'
import { formatPrice } from '@/core/utils/formatters'

export default function CheckoutPage() {
  const navigate = useNavigate()

  const items = useCartStore((s) => s.items)
  const subtotal = useCartStore((s) => s.subtotal())
  const itemCount = useCartStore((s) => s.itemCount())
  const isEmpty = useCartStore((s) => s.isEmpty())

  const placeOrder = useOrdersStore((s) => s.placeOrder)
  const isLoading = useOrdersStore((s) => s.isLoading)
  const error = useOrdersStore((s) => s.error)
  const clearError = useOrdersStore((s) => s.clearError)

  // Si el carrito está vacío al montar la página, redirige al carrito.
  useEffect(() => {
    if (isEmpty) {
      navigate('/cart', { replace: true })
    }
  }, [isEmpty, navigate])

  // Limpia el error del store al desmontar
  useEffect(() => {
    return () => {
      clearError()
    }
  }, [clearError])

  async function handleConfirm() {
    try {
      const newOrder = await placeOrder(items)
      navigate(`/orders/${newOrder.id}`, { replace: true })
    } catch {
      // El error ya está en el store; el componente lo muestra.
    }
  }

  if (isEmpty) return null

  return (
    <div className="container max-w-2xl py-12">
      <h1 className="mb-8 text-2xl font-bold">Confirmar pedido</h1>

      {/* Lista de ítems del carrito */}
      <section className="rounded-xl border">
        <div className="p-4">
          <h2 className="font-semibold text-sm uppercase tracking-wide text-muted-foreground">
            Resumen del pedido
          </h2>
        </div>
        <Separator />

        <ul className="divide-y">
          {items.map((item) => (
            <li
              key={item.product.id}
              className="flex items-center justify-between gap-4 px-4 py-3"
            >
              <div className="flex items-center gap-3">
                <div className="h-12 w-12 flex-shrink-0 overflow-hidden rounded-md border bg-muted">
                  {item.product.image && (
                    <img
                      src={item.product.image}
                      alt={item.product.name}
                      className="h-full w-full object-cover"
                    />
                  )}
                </div>
                <div className="text-sm">
                  <p className="font-medium leading-snug">{item.product.name}</p>
                  <p className="text-muted-foreground">
                    {item.quantity} × {formatPrice(Number(item.product.price))}
                  </p>
                </div>
              </div>
              <span className="text-sm font-semibold">
                {formatPrice(Number(item.product.price) * item.quantity)}
              </span>
            </li>
          ))}
        </ul>

        <Separator />

        <div className="flex justify-between px-4 py-3 text-sm">
          <span className="text-muted-foreground">
            {itemCount} {itemCount === 1 ? 'artículo' : 'artículos'}
          </span>
          <span className="font-bold text-base">{formatPrice(subtotal)}</span>
        </div>
      </section>

      {/* Error de la API */}
      {error && (
        <Alert variant="destructive" className="mt-6">
          <AlertCircle className="h-4 w-4" />
          <AlertDescription>{error}</AlertDescription>
        </Alert>
      )}

      {/* Acciones */}
      <div className="mt-8 flex flex-col gap-3 sm:flex-row-reverse">
        <Button
          size="lg"
          className="gap-2 sm:flex-1"
          onClick={handleConfirm}
          disabled={isLoading}
        >
          {isLoading ? (
            <>
              <Loader2 className="h-4 w-4 animate-spin" />
              Procesando…
            </>
          ) : (
            <>
              <ShoppingBag className="h-4 w-4" />
              Confirmar pedido
            </>
          )}
        </Button>

        <Button
          variant="outline"
          size="lg"
          className="sm:flex-1"
          asChild
          disabled={isLoading}
        >
          <Link to="/cart">Volver al carrito</Link>
        </Button>
      </div>
    </div>
  )
}
```

### Agregar la ruta al router

```tsx
// src/presentation/router/AppRouter.tsx  (fragmento — rutas protegidas)

import CheckoutPage from '@/presentation/pages/orders/CheckoutPage'
import OrdersPage from '@/presentation/pages/orders/OrdersPage'
import OrderDetailPage from '@/presentation/pages/orders/OrderDetailPage'

// Dentro de las rutas protegidas:
<Route path="orders/new" element={<CheckoutPage />} />
<Route path="orders" element={<OrdersPage />} />
<Route path="orders/:id" element={<OrderDetailPage />} />
```

---

## 7.5 OrdersPage

**Archivo:** `src/presentation/pages/orders/OrdersPage.tsx`

```tsx
// src/presentation/pages/orders/OrdersPage.tsx

import { useEffect } from 'react'
import { useNavigate } from 'react-router-dom'
import { Package } from 'lucide-react'

import { Card, CardContent, CardHeader } from '@/components/ui/card'
import { Skeleton } from '@/components/ui/skeleton'

import { useOrdersStore } from '@/domain/store/orders.store'
import { formatPrice, formatDate } from '@/core/utils/formatters'
import { StatusBadge } from '@/presentation/components/StatusBadge'

// ── Skeletons ────────────────────────────────────────────────────────────────

function OrderCardSkeleton() {
  return (
    <Card>
      <CardHeader className="pb-2">
        <div className="flex items-center justify-between">
          <Skeleton className="h-5 w-24" />
          <Skeleton className="h-5 w-20" />
        </div>
      </CardHeader>
      <CardContent className="flex items-center justify-between pt-0">
        <Skeleton className="h-4 w-32" />
        <Skeleton className="h-4 w-16" />
      </CardContent>
    </Card>
  )
}

// ── Estado vacío ─────────────────────────────────────────────────────────────

function EmptyOrders() {
  const navigate = useNavigate()
  return (
    <div className="flex flex-col items-center justify-center gap-6 py-24 text-center">
      <div className="flex h-20 w-20 items-center justify-center rounded-full bg-muted">
        <Package className="h-10 w-10 text-muted-foreground" />
      </div>
      <div>
        <h2 className="text-xl font-semibold">Sin pedidos todavía</h2>
        <p className="mt-1 text-sm text-muted-foreground">
          Cuando realices tu primer pedido aparecerá aquí.
        </p>
      </div>
      <button
        className="text-sm underline underline-offset-4 text-muted-foreground hover:text-foreground"
        onClick={() => navigate('/catalog')}
      >
        Ver catálogo
      </button>
    </div>
  )
}

// ── Página principal ─────────────────────────────────────────────────────────

export default function OrdersPage() {
  const navigate = useNavigate()
  const { orders, isLoading, fetchOrders } = useOrdersStore()

  useEffect(() => {
    fetchOrders()
  }, [fetchOrders])

  if (isLoading) {
    return (
      <div className="container py-8">
        <h1 className="mb-6 text-2xl font-bold">Mis pedidos</h1>
        <div className="space-y-4">
          {Array.from({ length: 4 }).map((_, i) => (
            <OrderCardSkeleton key={i} />
          ))}
        </div>
      </div>
    )
  }

  if (orders.length === 0) return <EmptyOrders />

  return (
    <div className="container py-8">
      <h1 className="mb-6 text-2xl font-bold">
        Mis pedidos{' '}
        <span className="text-base font-normal text-muted-foreground">
          ({orders.length})
        </span>
      </h1>

      <div className="space-y-4">
        {orders.map((order) => (
          <Card
            key={order.id}
            className="cursor-pointer transition-colors hover:bg-muted/50"
            onClick={() => navigate(`/orders/${order.id}`)}
            role="button"
            tabIndex={0}
            onKeyDown={(e) => {
              if (e.key === 'Enter' || e.key === ' ') {
                navigate(`/orders/${order.id}`)
              }
            }}
          >
            <CardHeader className="pb-2">
              <div className="flex items-center justify-between">
                {/* Id formateado como #0001 */}
                <span className="font-mono text-sm font-semibold">
                  #{String(order.id).padStart(4, '0')}
                </span>
                <StatusBadge status={order.status} />
              </div>
            </CardHeader>

            <CardContent className="flex items-center justify-between pt-0 text-sm text-muted-foreground">
              <div className="flex items-center gap-4">
                <span>{formatDate(order.created_at)}</span>
                <span>
                  {order.total_items}{' '}
                  {order.total_items === 1 ? 'artículo' : 'artículos'}
                </span>
              </div>
              <span className="font-semibold text-foreground">
                {formatPrice(Number(order.total))}
              </span>
            </CardContent>
          </Card>
        ))}
      </div>
    </div>
  )
}
```

Cada `Card` es navegable con teclado (`tabIndex`, `onKeyDown`) para cumplir con accesibilidad básica
sin usar un elemento `<a>` que requeriría una URL como href.

---

## 7.6 OrderDetailPage

**Archivo:** `src/presentation/pages/orders/OrderDetailPage.tsx`

```tsx
// src/presentation/pages/orders/OrderDetailPage.tsx

import { useEffect } from 'react'
import { useParams, Link } from 'react-router-dom'
import { ArrowLeft } from 'lucide-react'

import { Button } from '@/components/ui/button'
import { Separator } from '@/components/ui/separator'
import { Skeleton } from '@/components/ui/skeleton'
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table'

import { useOrdersStore } from '@/domain/store/orders.store'
import { formatPrice, formatDate } from '@/core/utils/formatters'
import { StatusBadge } from '@/presentation/components/StatusBadge'
import type { OrderStatus } from '@/domain/model/orders.types'

// ── Línea de tiempo de estado ─────────────────────────────────────────────────

const STATUS_STEPS: { status: OrderStatus; label: string }[] = [
  { status: 'pending', label: 'Recibido' },
  { status: 'confirmed', label: 'Confirmado' },
  { status: 'shipped', label: 'Enviado' },
  { status: 'delivered', label: 'Entregado' },
]

interface StatusTimelineProps {
  currentStatus: OrderStatus
}

function StatusTimeline({ currentStatus }: StatusTimelineProps) {
  if (currentStatus === 'cancelled') {
    return (
      <div className="flex items-center gap-2 text-sm text-destructive">
        <span className="font-medium">Pedido cancelado</span>
      </div>
    )
  }

  const currentIndex = STATUS_STEPS.findIndex(
    (s) => s.status === currentStatus,
  )

  return (
    <ol className="flex items-center gap-0 overflow-x-auto">
      {STATUS_STEPS.map((step, index) => {
        const isDone = index <= currentIndex
        const isLast = index === STATUS_STEPS.length - 1

        return (
          <li key={step.status} className="flex items-center">
            <div className="flex flex-col items-center gap-1">
              <div
                className={[
                  'flex h-8 w-8 items-center justify-center rounded-full text-xs font-bold',
                  isDone
                    ? 'bg-primary text-primary-foreground'
                    : 'border-2 border-muted text-muted-foreground',
                ].join(' ')}
              >
                {index + 1}
              </div>
              <span
                className={[
                  'whitespace-nowrap text-xs',
                  isDone ? 'text-primary font-medium' : 'text-muted-foreground',
                ].join(' ')}
              >
                {step.label}
              </span>
            </div>

            {!isLast && (
              <div
                className={[
                  'mx-2 h-0.5 w-12 flex-shrink-0',
                  index < currentIndex ? 'bg-primary' : 'bg-muted',
                ].join(' ')}
              />
            )}
          </li>
        )
      })}
    </ol>
  )
}

// ── Skeleton ──────────────────────────────────────────────────────────────────

function OrderDetailSkeleton() {
  return (
    <div className="container max-w-3xl py-8">
      <Skeleton className="mb-6 h-6 w-32" />
      <div className="mb-6 flex items-center justify-between">
        <Skeleton className="h-8 w-40" />
        <Skeleton className="h-6 w-24" />
      </div>
      <Skeleton className="mb-8 h-12 w-full" />
      <Skeleton className="h-48 w-full rounded-xl" />
    </div>
  )
}

// ── Página principal ─────────────────────────────────────────────────────────

export default function OrderDetailPage() {
  const { id } = useParams<{ id: string }>()
  const { currentOrder, isLoading, error, fetchOrderById } = useOrdersStore()

  useEffect(() => {
    if (id) fetchOrderById(Number(id))
  }, [id, fetchOrderById])

  if (isLoading) return <OrderDetailSkeleton />

  if (error || !currentOrder) {
    return (
      <div className="container py-16 text-center">
        <p className="text-muted-foreground">
          {error ?? 'Orden no encontrada.'}
        </p>
        <Button asChild className="mt-4" variant="outline">
          <Link to="/orders">Ver mis pedidos</Link>
        </Button>
      </div>
    )
  }

  const order = currentOrder

  return (
    <div className="container max-w-3xl py-8">
      {/* Botón volver */}
      <Button asChild variant="ghost" size="sm" className="-ml-2 mb-4 gap-1">
        <Link to="/orders">
          <ArrowLeft className="h-4 w-4" />
          Mis pedidos
        </Link>
      </Button>

      {/* Encabezado */}
      <div className="mb-6 flex flex-wrap items-start justify-between gap-4">
        <div>
          <h1 className="font-mono text-2xl font-bold">
            Pedido #{String(order.id).padStart(4, '0')}
          </h1>
          <p className="mt-1 text-sm text-muted-foreground">
            Realizado el {formatDate(order.created_at)}
            {order.updated_at !== order.created_at && (
              <> · Actualizado el {formatDate(order.updated_at)}</>
            )}
          </p>
        </div>
        <StatusBadge status={order.status} />
      </div>

      {/* Línea de tiempo */}
      <div className="mb-8 rounded-xl border p-4">
        <StatusTimeline currentStatus={order.status} />
      </div>

      {/* Tabla de ítems */}
      <div className="rounded-xl border">
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Producto</TableHead>
              <TableHead className="text-right">Cant.</TableHead>
              <TableHead className="text-right">Precio unit.</TableHead>
              <TableHead className="text-right">Subtotal</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {order.items.map((item) => (
              <TableRow key={item.id}>
                <TableCell className="font-medium">
                  {item.product_name}
                </TableCell>
                <TableCell className="text-right tabular-nums">
                  {item.quantity}
                </TableCell>
                <TableCell className="text-right tabular-nums">
                  {formatPrice(Number(item.unit_price))}
                </TableCell>
                <TableCell className="text-right tabular-nums">
                  {formatPrice(Number(item.subtotal))}
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>

        <Separator />

        {/* Resumen de totales */}
        <div className="space-y-2 p-4">
          <div className="flex justify-between text-sm text-muted-foreground">
            <span>
              Total de artículos:{' '}
              <span className="font-medium text-foreground">
                {order.total_items}
              </span>
            </span>
          </div>
          <div className="flex justify-between text-base font-bold">
            <span>Total del pedido</span>
            <span>{formatPrice(Number(order.total))}</span>
          </div>
        </div>
      </div>
    </div>
  )
}
```

El componente `StatusTimeline` maneja el caso de cancelación por separado porque `cancelled` no
tiene una posición lineal en el flujo.

---

## 7.7 StatusBadge

**Archivo:** `src/presentation/components/StatusBadge.tsx`

```tsx
// src/presentation/components/StatusBadge.tsx

import { Badge } from '@/components/ui/badge'
import type { OrderStatus } from '@/domain/model/orders.types'

interface StatusConfig {
  label: string
  className: string
}

/**
 * Mapa de configuración de cada estado. Usa clases de Tailwind en lugar de
 * variantes de shadcn para poder definir colores semánticos exactos.
 */
const STATUS_MAP: Record<OrderStatus, StatusConfig> = {
  pending: {
    label: 'Pendiente',
    className:
      'bg-yellow-100 text-yellow-800 hover:bg-yellow-100 border-yellow-200',
  },
  confirmed: {
    label: 'Confirmado',
    className: 'bg-blue-100 text-blue-800 hover:bg-blue-100 border-blue-200',
  },
  shipped: {
    label: 'Enviado',
    className:
      'bg-purple-100 text-purple-800 hover:bg-purple-100 border-purple-200',
  },
  delivered: {
    label: 'Entregado',
    className:
      'bg-green-100 text-green-800 hover:bg-green-100 border-green-200',
  },
  cancelled: {
    label: 'Cancelado',
    className: 'bg-red-100 text-red-800 hover:bg-red-100 border-red-200',
  },
}

interface StatusBadgeProps {
  status: OrderStatus
  className?: string
}

/**
 * Badge reutilizable que traduce un `OrderStatus` a etiqueta y color.
 * Se usa en `OrdersPage` (listado) y `OrderDetailPage` (detalle).
 */
export function StatusBadge({ status, className = '' }: StatusBadgeProps) {
  const config = STATUS_MAP[status]

  return (
    <Badge
      variant="outline"
      className={[config.className, className].filter(Boolean).join(' ')}
    >
      {config.label}
    </Badge>
  )
}
```

Se usa `variant="outline"` de shadcn como base y se sobreescriben los colores con clases de
Tailwind. El `hover:bg-*` mantiene el mismo color que el fondo para que el badge no cambie al pasar
el ratón por encima (los badges no son interactivos).

---

## 7.8 Checkpoint y tabla resumen

Antes de continuar con el módulo 8, verifica el flujo completo:

1. Agrega productos al carrito desde `/catalog/:id`.
2. En `/cart` haz clic en "Proceder al pago" → llegas a `/orders/new`.
3. Haz clic en "Confirmar pedido" → mientras procesa aparece el spinner.
4. Al completar, la app navega a `/orders/:id` con el detalle de la orden nueva.
5. El carrito queda vacío (badge en `0` y localStorage limpio).
6. En `/orders` aparece la orden con el badge "Pendiente" en amarillo.
7. Haz clic en la card → llegas al detalle con la tabla de ítems y la línea de tiempo.
8. Verifica que `StatusBadge` muestra el color correcto para cada estado.

### Tabla resumen del módulo 7

| Archivo | Capa | Descripción |
|---|---|---|
| `src/domain/model/orders.types.ts` | Dominio | `OrderStatus`, `Order`, `OrderItem`, `CreateOrderPayload` |
| `src/data/api/orders.api.ts` | Datos | `getOrders`, `getOrderById`, `createOrder` con `apiClient` |
| `src/domain/store/orders.store.ts` | Dominio | Store Zustand: `fetchOrders`, `fetchOrderById`, `placeOrder` |
| `src/presentation/pages/orders/CheckoutPage.tsx` | Presentación | Resumen del pedido y confirmación |
| `src/presentation/pages/orders/OrdersPage.tsx` | Presentación | Listado de órdenes con cards y badges |
| `src/presentation/pages/orders/OrderDetailPage.tsx` | Presentación | Detalle con tabla de ítems y línea de tiempo |
| `src/presentation/components/StatusBadge.tsx` | Presentación | Badge de estado con colores semánticos |
