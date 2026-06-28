# ShopApp React Web — Módulo 6

## Carrito — CartStore, ProductDetail completo y CartPage

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulos 1–5 completados.

---

> **Objetivo**
> Implementar el módulo de carrito de compras de extremo a extremo: tipos de dominio, store Zustand
> persistido en localStorage, icono de carrito en el header, página de detalle de producto con
> selector de cantidad y botón "Agregar al carrito", página de carrito con resumen y controles de
> cantidad, y un CartDrawer opcional como panel lateral.
>
> **Checkpoint final**
> - `cart.store.ts` persiste los ítems en localStorage y recalcula `itemCount` y `subtotal` automáticamente.
> - `ProductDetailPage` carga el producto por id, muestra el selector de cantidad y agrega al carrito.
> - Aparece un toast de confirmación al agregar un producto.
> - `CartPage` lista los ítems con controles de cantidad, totales y botón de checkout.
> - `CartIcon` en el header muestra el badge con la cantidad de ítems.
> - `CartDrawer` (opcional) abre un panel lateral al hacer clic en el icono.

---

## 6.1 Tipos del dominio del carrito

**Archivo:** `src/domain/model/cart.types.ts`

```typescript
// src/domain/model/cart.types.ts

import type { Product } from './catalog.types'

/**
 * Un ítem dentro del carrito: el producto completo más la cantidad elegida.
 */
export interface CartItem {
  product: Product
  quantity: number
}

/**
 * Estado completo del store del carrito.
 */
export interface CartState {
  items: CartItem[]
  isOpen: boolean
}

/**
 * Resumen calculado del carrito, útil para mostrar en el header o sidebar.
 */
export interface CartSummary {
  /** Precio total sin descuentos ni impuestos adicionales. */
  subtotal: number
  /** Suma de las cantidades de todos los ítems. */
  itemCount: number
}
```

`CartItem` agrupa el objeto `Product` completo (importado de `catalog.types.ts`) junto con la
cantidad. Esto permite mostrar nombre, imagen y precio directamente desde el store sin volver a
llamar a la API. `CartState` es la forma del estado persistido y `CartSummary` es el tipo de
retorno de los valores derivados.

---

## 6.2 CartStore

**Archivo:** `src/domain/store/cart.store.ts`

```typescript
// src/domain/store/cart.store.ts

import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'
import type { CartItem } from '../model/cart.types'
import type { Product } from '../model/catalog.types'

interface CartStore {
  // ── Estado ──────────────────────────────────────────────────────────────
  items: CartItem[]
  isOpen: boolean

  // ── Valores derivados ────────────────────────────────────────────────────
  itemCount: () => number
  subtotal: () => number
  isEmpty: () => boolean

  // ── Acciones ─────────────────────────────────────────────────────────────
  addItem: (product: Product, quantity?: number) => void
  removeItem: (productId: number) => void
  updateQuantity: (productId: number, quantity: number) => void
  clearCart: () => void
  toggleCart: () => void
  openCart: () => void
  closeCart: () => void
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      // ── Estado inicial ────────────────────────────────────────────────────
      items: [],
      isOpen: false,

      // ── Valores derivados ─────────────────────────────────────────────────
      itemCount: () =>
        get().items.reduce((acc, item) => acc + item.quantity, 0),

      subtotal: () =>
        get().items.reduce(
          (acc, item) => acc + Number(item.product.price) * item.quantity,
          0,
        ),

      isEmpty: () => get().items.length === 0,

      // ── Acciones ──────────────────────────────────────────────────────────

      /**
       * Agrega un producto al carrito. Si ya existe, incrementa su cantidad.
       * La cantidad por defecto es 1.
       */
      addItem: (product, quantity = 1) => {
        set((state) => {
          const existing = state.items.find(
            (item) => item.product.id === product.id,
          )

          if (existing) {
            return {
              items: state.items.map((item) =>
                item.product.id === product.id
                  ? { ...item, quantity: item.quantity + quantity }
                  : item,
              ),
            }
          }

          return { items: [...state.items, { product, quantity }] }
        })
      },

      /**
       * Elimina completamente un ítem del carrito por id de producto.
       */
      removeItem: (productId) => {
        set((state) => ({
          items: state.items.filter((item) => item.product.id !== productId),
        }))
      },

      /**
       * Actualiza la cantidad de un ítem. Si quantity llega a 0, elimina el ítem.
       */
      updateQuantity: (productId, quantity) => {
        if (quantity <= 0) {
          get().removeItem(productId)
          return
        }

        set((state) => ({
          items: state.items.map((item) =>
            item.product.id === productId ? { ...item, quantity } : item,
          ),
        }))
      },

      /** Vacía el carrito por completo. */
      clearCart: () => set({ items: [] }),

      /** Alterna la visibilidad del CartDrawer. */
      toggleCart: () => set((state) => ({ isOpen: !state.isOpen })),

      /** Abre el CartDrawer. */
      openCart: () => set({ isOpen: true }),

      /** Cierra el CartDrawer. */
      closeCart: () => set({ isOpen: false }),
    }),
    {
      name: 'shopapp_cart',
      storage: createJSONStorage(() => localStorage),
      // Solo persistimos los ítems; isOpen siempre arranca en false.
      partialize: (state) => ({ items: state.items }),
    },
  ),
)
```

El middleware `persist` de Zustand serializa el estado seleccionado por `partialize` y lo guarda en
`localStorage` bajo la clave `shopapp_cart`. Al recargar la página, los ítems se restauran
automáticamente. `isOpen` no se persiste para que el drawer siempre arranque cerrado.

Los valores derivados `itemCount`, `subtotal` e `isEmpty` se definen como funciones que llaman a
`get()` en lugar de guardarse como propiedades de estado. Esto evita dobles actualizaciones de
React: se recalculan en cada render que use el store.

---

## 6.3 CartIcon

**Archivo:** `src/presentation/components/CartIcon.tsx`

```tsx
// src/presentation/components/CartIcon.tsx

import { ShoppingCart } from 'lucide-react'
import { useNavigate } from 'react-router-dom'
import { Button } from '@/components/ui/button'
import { useCartStore } from '@/domain/store/cart.store'

export function CartIcon() {
  const navigate = useNavigate()
  const itemCount = useCartStore((s) => s.itemCount())

  return (
    <Button
      variant="ghost"
      size="icon"
      aria-label={`Carrito, ${itemCount} productos`}
      className="relative"
      onClick={() => navigate('/cart')}
    >
      <ShoppingCart className="h-5 w-5" />

      {itemCount > 0 && (
        <span
          className={[
            'absolute -right-1 -top-1 flex h-5 w-5 items-center justify-center',
            'rounded-full bg-primary text-[10px] font-bold text-primary-foreground',
          ].join(' ')}
        >
          {itemCount > 99 ? '99+' : itemCount}
        </span>
      )}
    </Button>
  )
}
```

### Integración en AppShell

Abre `src/presentation/components/AppShell.tsx` y agrega `CartIcon` en el header:

```tsx
// src/presentation/components/AppShell.tsx  (fragmento — sección del header)

import { CartIcon } from './CartIcon'

// Dentro del <header>:
<header className="sticky top-0 z-40 w-full border-b bg-background">
  <div className="container flex h-14 items-center justify-between">
    {/* Logo / navegación izquierda */}
    <nav className="flex items-center gap-4">
      <Link to="/" className="font-bold text-lg tracking-tight">
        ShopApp
      </Link>
      <Link
        to="/catalog"
        className="text-sm text-muted-foreground hover:text-foreground transition-colors"
      >
        Catálogo
      </Link>
    </nav>

    {/* Acciones del header */}
    <div className="flex items-center gap-2">
      <CartIcon />
      {/* otros botones como UserMenu */}
    </div>
  </div>
</header>
```

El badge solo aparece cuando `itemCount > 0`. Para cuentas con muchos ítems, se muestra `99+` para
no desbordar el badge.

---

## 6.4 ProductDetailPage (completo)

**Archivo:** `src/presentation/pages/catalog/ProductDetailPage.tsx`

```tsx
// src/presentation/pages/catalog/ProductDetailPage.tsx

import { useEffect, useState } from 'react'
import { useParams, Link } from 'react-router-dom'
import { Minus, Plus, ShoppingCart } from 'lucide-react'
import { toast } from 'sonner'

import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'
import { Skeleton } from '@/components/ui/skeleton'
import {
  Breadcrumb,
  BreadcrumbItem,
  BreadcrumbLink,
  BreadcrumbList,
  BreadcrumbPage,
  BreadcrumbSeparator,
} from '@/components/ui/breadcrumb'

import { getProductById } from '@/data/api/catalog.api'
import { useCartStore } from '@/domain/store/cart.store'
import { formatPrice } from '@/core/utils/formatters'
import type { Product } from '@/domain/model/catalog.types'

// ── Skeleton de carga ────────────────────────────────────────────────────────

function ProductDetailSkeleton() {
  return (
    <div className="container py-8">
      <Skeleton className="mb-6 h-5 w-64" />
      <div className="grid gap-8 md:grid-cols-2">
        <Skeleton className="aspect-square w-full rounded-xl" />
        <div className="flex flex-col gap-4">
          <Skeleton className="h-8 w-3/4" />
          <Skeleton className="h-5 w-24" />
          <Skeleton className="h-10 w-32" />
          <Skeleton className="h-4 w-full" />
          <Skeleton className="h-4 w-5/6" />
          <Skeleton className="h-4 w-4/6" />
          <div className="mt-4 flex gap-3">
            <Skeleton className="h-10 w-32" />
            <Skeleton className="h-10 flex-1" />
          </div>
        </div>
      </div>
    </div>
  )
}

// ── Selector de cantidad ─────────────────────────────────────────────────────

interface QuantitySelectorProps {
  value: number
  min?: number
  max: number
  onChange: (value: number) => void
}

function QuantitySelector({
  value,
  min = 1,
  max,
  onChange,
}: QuantitySelectorProps) {
  return (
    <div className="flex items-center rounded-md border">
      <Button
        variant="ghost"
        size="icon"
        className="h-10 w-10 rounded-none rounded-l-md"
        onClick={() => onChange(Math.max(min, value - 1))}
        disabled={value <= min}
        aria-label="Reducir cantidad"
      >
        <Minus className="h-4 w-4" />
      </Button>

      <span className="flex h-10 w-12 items-center justify-center text-sm font-medium tabular-nums">
        {value}
      </span>

      <Button
        variant="ghost"
        size="icon"
        className="h-10 w-10 rounded-none rounded-r-md"
        onClick={() => onChange(Math.min(max, value + 1))}
        disabled={value >= max}
        aria-label="Aumentar cantidad"
      >
        <Plus className="h-4 w-4" />
      </Button>
    </div>
  )
}

// ── Página principal ─────────────────────────────────────────────────────────

export default function ProductDetailPage() {
  const { id } = useParams<{ id: string }>()
  const addItem = useCartStore((s) => s.addItem)

  const [product, setProduct] = useState<Product | null>(null)
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
  const [quantity, setQuantity] = useState(1)

  useEffect(() => {
    if (!id) return

    setIsLoading(true)
    setError(null)

    getProductById(Number(id))
      .then((data) => {
        setProduct(data)
        setQuantity(1)
      })
      .catch(() => setError('No se pudo cargar el producto.'))
      .finally(() => setIsLoading(false))
  }, [id])

  if (isLoading) return <ProductDetailSkeleton />

  if (error || !product) {
    return (
      <div className="container py-16 text-center">
        <p className="text-muted-foreground">{error ?? 'Producto no encontrado.'}</p>
        <Link to="/catalog">
          <Button className="mt-4" variant="outline">
            Volver al catálogo
          </Button>
        </Link>
      </div>
    )
  }

  const inStock = Number(product.stock) > 0

  function handleAddToCart() {
    if (!product) return
    addItem(product, quantity)
    toast.success('Producto agregado al carrito', {
      description: `${quantity} × ${product.name}`,
      duration: 3000,
    })
  }

  return (
    <div className="container py-8">
      {/* Breadcrumb */}
      <Breadcrumb className="mb-6">
        <BreadcrumbList>
          <BreadcrumbItem>
            <BreadcrumbLink asChild>
              <Link to="/">Inicio</Link>
            </BreadcrumbLink>
          </BreadcrumbItem>
          <BreadcrumbSeparator />
          <BreadcrumbItem>
            <BreadcrumbLink asChild>
              <Link to="/catalog">Catálogo</Link>
            </BreadcrumbLink>
          </BreadcrumbItem>
          <BreadcrumbSeparator />
          <BreadcrumbItem>
            <BreadcrumbPage>{product.name}</BreadcrumbPage>
          </BreadcrumbItem>
        </BreadcrumbList>
      </Breadcrumb>

      {/* Layout de dos columnas */}
      <div className="grid gap-8 md:grid-cols-2">
        {/* Imagen del producto */}
        <div className="overflow-hidden rounded-xl border bg-muted">
          {product.image ? (
            <img
              src={product.image}
              alt={product.name}
              className="aspect-square w-full object-cover"
              onError={(e) => {
                ;(e.currentTarget as HTMLImageElement).src =
                  '/placeholder-product.png'
              }}
            />
          ) : (
            <div className="flex aspect-square w-full items-center justify-center text-muted-foreground">
              <ShoppingCart className="h-24 w-24 opacity-20" />
            </div>
          )}
        </div>

        {/* Información del producto */}
        <div className="flex flex-col gap-5">
          {/* Categoría */}
          {product.category && (
            <Badge variant="secondary" className="w-fit">
              {product.category.name}
            </Badge>
          )}

          {/* Nombre */}
          <h1 className="text-3xl font-bold leading-tight">{product.name}</h1>

          {/* Precio */}
          <p className="text-4xl font-extrabold tracking-tight text-primary">
            {formatPrice(Number(product.price))}
          </p>

          {/* Stock */}
          <div className="flex items-center gap-2 text-sm">
            <span
              className={[
                'inline-block h-2 w-2 rounded-full',
                inStock ? 'bg-green-500' : 'bg-red-500',
              ].join(' ')}
            />
            <span className={inStock ? 'text-green-700' : 'text-red-600'}>
              {inStock
                ? `${product.stock} unidades disponibles`
                : 'Agotado'}
            </span>
          </div>

          {/* Descripción */}
          {product.description && (
            <p className="text-sm leading-relaxed text-muted-foreground">
              {product.description}
            </p>
          )}

          {/* Selector de cantidad y botón */}
          <div className="flex flex-wrap items-center gap-3 pt-2">
            <QuantitySelector
              value={quantity}
              min={1}
              max={Number(product.stock)}
              onChange={setQuantity}
            />

            <Button
              className="flex-1 gap-2"
              size="lg"
              disabled={!inStock}
              onClick={handleAddToCart}
            >
              <ShoppingCart className="h-5 w-5" />
              {inStock ? 'Agregar al carrito' : 'Agotado'}
            </Button>
          </div>
        </div>
      </div>
    </div>
  )
}
```

La función `handleAddToCart` llama a `addItem` del store y dispara un toast de Sonner con el nombre
del producto y la cantidad. El componente `QuantitySelector` es interno a este archivo; si se
necesita en otros lugares se puede extraer a `src/presentation/components/QuantitySelector.tsx`.

El skeleton ocupa exactamente el mismo espacio que el contenido real para evitar el salto de layout
(Cumulative Layout Shift).

---

## 6.5 CartPage

**Archivo:** `src/presentation/pages/cart/CartPage.tsx`

```tsx
// src/presentation/pages/cart/CartPage.tsx

import { Link, useNavigate } from 'react-router-dom'
import { Minus, Plus, ShoppingCart, Trash2 } from 'lucide-react'

import { Button } from '@/components/ui/button'
import { Separator } from '@/components/ui/separator'

import { useCartStore } from '@/domain/store/cart.store'
import { formatPrice } from '@/core/utils/formatters'
import type { CartItem } from '@/domain/model/cart.types'

// ── Fila de un ítem ──────────────────────────────────────────────────────────

interface CartItemRowProps {
  item: CartItem
  onUpdateQuantity: (productId: number, quantity: number) => void
  onRemove: (productId: number) => void
}

function CartItemRow({ item, onUpdateQuantity, onRemove }: CartItemRowProps) {
  const { product, quantity } = item
  const lineTotal = Number(product.price) * quantity

  return (
    <>
      <div className="flex gap-4 py-4">
        {/* Miniatura */}
        <div className="h-20 w-20 flex-shrink-0 overflow-hidden rounded-md border bg-muted">
          {product.image ? (
            <img
              src={product.image}
              alt={product.name}
              className="h-full w-full object-cover"
              onError={(e) => {
                ;(e.currentTarget as HTMLImageElement).src =
                  '/placeholder-product.png'
              }}
            />
          ) : (
            <div className="flex h-full w-full items-center justify-center">
              <ShoppingCart className="h-8 w-8 text-muted-foreground/40" />
            </div>
          )}
        </div>

        {/* Detalles */}
        <div className="flex flex-1 flex-col justify-between">
          <div className="flex items-start justify-between gap-2">
            <div>
              <Link
                to={`/catalog/${product.id}`}
                className="font-medium hover:underline"
              >
                {product.name}
              </Link>
              <p className="mt-0.5 text-sm text-muted-foreground">
                {formatPrice(Number(product.price))} / ud.
              </p>
            </div>

            {/* Eliminar */}
            <Button
              variant="ghost"
              size="icon"
              className="h-8 w-8 text-muted-foreground hover:text-destructive"
              onClick={() => onRemove(product.id)}
              aria-label={`Eliminar ${product.name}`}
            >
              <Trash2 className="h-4 w-4" />
            </Button>
          </div>

          {/* Controles de cantidad y total de línea */}
          <div className="flex items-center justify-between">
            <div className="flex items-center rounded-md border">
              <Button
                variant="ghost"
                size="icon"
                className="h-8 w-8 rounded-none rounded-l-md"
                onClick={() => onUpdateQuantity(product.id, quantity - 1)}
                disabled={quantity <= 1}
                aria-label="Reducir cantidad"
              >
                <Minus className="h-3 w-3" />
              </Button>

              <span className="flex h-8 w-10 items-center justify-center text-sm font-medium tabular-nums">
                {quantity}
              </span>

              <Button
                variant="ghost"
                size="icon"
                className="h-8 w-8 rounded-none rounded-r-md"
                onClick={() =>
                  onUpdateQuantity(product.id, quantity + 1)
                }
                disabled={quantity >= Number(product.stock)}
                aria-label="Aumentar cantidad"
              >
                <Plus className="h-3 w-3" />
              </Button>
            </div>

            <span className="font-semibold">{formatPrice(lineTotal)}</span>
          </div>
        </div>
      </div>

      <Separator />
    </>
  )
}

// ── Estado vacío ─────────────────────────────────────────────────────────────

function EmptyCart() {
  return (
    <div className="flex flex-col items-center justify-center gap-6 py-24 text-center">
      <div className="flex h-20 w-20 items-center justify-center rounded-full bg-muted">
        <ShoppingCart className="h-10 w-10 text-muted-foreground" />
      </div>
      <div>
        <h2 className="text-xl font-semibold">Tu carrito está vacío</h2>
        <p className="mt-1 text-sm text-muted-foreground">
          Agrega productos para verlos aquí.
        </p>
      </div>
      <Link to="/catalog">
        <Button>Ver catálogo</Button>
      </Link>
    </div>
  )
}

// ── Página principal ─────────────────────────────────────────────────────────

export default function CartPage() {
  const navigate = useNavigate()

  const items = useCartStore((s) => s.items)
  const itemCount = useCartStore((s) => s.itemCount())
  const subtotal = useCartStore((s) => s.subtotal())
  const updateQuantity = useCartStore((s) => s.updateQuantity)
  const removeItem = useCartStore((s) => s.removeItem)
  const isEmpty = useCartStore((s) => s.isEmpty())

  if (isEmpty) return <EmptyCart />

  return (
    <div className="container py-8">
      <h1 className="mb-8 text-2xl font-bold">
        Carrito{' '}
        <span className="text-muted-foreground font-normal text-base">
          ({itemCount} {itemCount === 1 ? 'producto' : 'productos'})
        </span>
      </h1>

      <div className="grid gap-8 lg:grid-cols-3">
        {/* Lista de ítems */}
        <div className="lg:col-span-2">
          {items.map((item) => (
            <CartItemRow
              key={item.product.id}
              item={item}
              onUpdateQuantity={updateQuantity}
              onRemove={removeItem}
            />
          ))}

          <div className="pt-4">
            <Link
              to="/catalog"
              className="text-sm text-muted-foreground hover:text-foreground underline underline-offset-4"
            >
              ← Continuar comprando
            </Link>
          </div>
        </div>

        {/* Resumen — sticky en desktop */}
        <aside className="h-fit rounded-xl border p-6 lg:sticky lg:top-24">
          <h2 className="mb-4 text-lg font-semibold">Resumen del pedido</h2>

          <div className="space-y-3 text-sm">
            <div className="flex justify-between text-muted-foreground">
              <span>
                Subtotal ({itemCount} {itemCount === 1 ? 'artículo' : 'artículos'})
              </span>
              <span>{formatPrice(subtotal)}</span>
            </div>
            <Separator />
            <div className="flex justify-between text-base font-bold">
              <span>Total</span>
              <span>{formatPrice(subtotal)}</span>
            </div>
          </div>

          <Button
            className="mt-6 w-full"
            size="lg"
            onClick={() => navigate('/orders/new')}
          >
            Proceder al pago
          </Button>

          <p className="mt-3 text-center text-xs text-muted-foreground">
            Los impuestos y gastos de envío se calculan en el siguiente paso.
          </p>
        </aside>
      </div>
    </div>
  )
}
```

`CartPage` es una ruta protegida: si el usuario no está autenticado, el guardia de ruta del módulo
2 lo redirige a `/login`. La ruta `/orders/new` se implementa en el módulo 7.

El sidebar con `lg:sticky lg:top-24` permanece visible mientras se hace scroll por la lista de
ítems en pantallas grandes.

---

## 6.6 CartDrawer (opcional)

**Archivo:** `src/presentation/components/CartDrawer.tsx`

```tsx
// src/presentation/components/CartDrawer.tsx

import { Link, useNavigate } from 'react-router-dom'
import { ShoppingCart, X } from 'lucide-react'

import {
  Sheet,
  SheetContent,
  SheetHeader,
  SheetTitle,
  SheetFooter,
} from '@/components/ui/sheet'
import { Button } from '@/components/ui/button'
import { Separator } from '@/components/ui/separator'
import { ScrollArea } from '@/components/ui/scroll-area'

import { useCartStore } from '@/domain/store/cart.store'
import { formatPrice } from '@/core/utils/formatters'

export function CartDrawer() {
  const navigate = useNavigate()

  const isOpen = useCartStore((s) => s.isOpen)
  const closeCart = useCartStore((s) => s.closeCart)
  const items = useCartStore((s) => s.items)
  const itemCount = useCartStore((s) => s.itemCount())
  const subtotal = useCartStore((s) => s.subtotal())
  const isEmpty = useCartStore((s) => s.isEmpty())

  function handleGoToCart() {
    closeCart()
    navigate('/cart')
  }

  function handleCheckout() {
    closeCart()
    navigate('/orders/new')
  }

  return (
    <Sheet open={isOpen} onOpenChange={(open) => !open && closeCart()}>
      <SheetContent side="right" className="flex w-full flex-col sm:max-w-sm">
        <SheetHeader>
          <SheetTitle className="flex items-center gap-2">
            <ShoppingCart className="h-5 w-5" />
            Carrito{' '}
            {itemCount > 0 && (
              <span className="text-sm font-normal text-muted-foreground">
                ({itemCount})
              </span>
            )}
          </SheetTitle>
        </SheetHeader>

        {isEmpty ? (
          <div className="flex flex-1 flex-col items-center justify-center gap-4 text-center">
            <ShoppingCart className="h-12 w-12 text-muted-foreground/30" />
            <p className="text-sm text-muted-foreground">
              Tu carrito está vacío.
            </p>
            <Button variant="outline" onClick={closeCart} asChild>
              <Link to="/catalog" onClick={closeCart}>
                Ver catálogo
              </Link>
            </Button>
          </div>
        ) : (
          <>
            {/* Lista de ítems con scroll */}
            <ScrollArea className="flex-1 py-4">
              <ul className="space-y-4 pr-4">
                {items.map((item) => (
                  <li key={item.product.id} className="flex gap-3">
                    <div className="h-16 w-16 flex-shrink-0 overflow-hidden rounded-md border bg-muted">
                      {item.product.image ? (
                        <img
                          src={item.product.image}
                          alt={item.product.name}
                          className="h-full w-full object-cover"
                        />
                      ) : (
                        <div className="flex h-full w-full items-center justify-center">
                          <ShoppingCart className="h-6 w-6 text-muted-foreground/30" />
                        </div>
                      )}
                    </div>

                    <div className="flex flex-1 flex-col justify-between text-sm">
                      <span className="line-clamp-2 font-medium leading-snug">
                        {item.product.name}
                      </span>
                      <div className="flex items-center justify-between text-muted-foreground">
                        <span>
                          {item.quantity} × {formatPrice(Number(item.product.price))}
                        </span>
                        <span className="font-semibold text-foreground">
                          {formatPrice(Number(item.product.price) * item.quantity)}
                        </span>
                      </div>
                    </div>
                  </li>
                ))}
              </ul>
            </ScrollArea>

            <Separator />

            {/* Subtotal */}
            <div className="flex justify-between py-3 text-sm font-semibold">
              <span>Subtotal</span>
              <span>{formatPrice(subtotal)}</span>
            </div>

            <SheetFooter className="flex-col gap-2 sm:flex-col">
              <Button className="w-full" onClick={handleCheckout}>
                Ir al pago
              </Button>
              <Button variant="outline" className="w-full" onClick={handleGoToCart}>
                Ver carrito completo
              </Button>
              <Button variant="ghost" className="w-full gap-2" onClick={closeCart}>
                <X className="h-4 w-4" />
                Cerrar
              </Button>
            </SheetFooter>
          </>
        )}
      </SheetContent>
    </Sheet>
  )
}
```

### Integración del CartDrawer en AppShell

Para usar el drawer en lugar de (o además de) navegar a `/cart`, agrega `CartDrawer` al final del
componente `AppShell`:

```tsx
// src/presentation/components/AppShell.tsx  (fragmento)

import { CartDrawer } from './CartDrawer'

export function AppShell({ children }: { children: React.ReactNode }) {
  return (
    <>
      <header>...</header>
      <main className="container py-6">{children}</main>
      <CartDrawer />
    </>
  )
}
```

Y cambia el `onClick` de `CartIcon` para abrir el drawer en vez de navegar:

```tsx
// src/presentation/components/CartIcon.tsx  (variante con drawer)

import { useCartStore } from '@/domain/store/cart.store'

export function CartIcon() {
  const itemCount = useCartStore((s) => s.itemCount())
  const openCart = useCartStore((s) => s.openCart)

  return (
    <Button
      variant="ghost"
      size="icon"
      aria-label={`Carrito, ${itemCount} productos`}
      className="relative"
      onClick={openCart}   // ← abre el drawer en lugar de navegar
    >
      ...
    </Button>
  )
}
```

Con este enfoque el drawer funciona como acceso rápido y `/cart` sigue disponible como página
completa.

---

## 6.7 Registro de rutas

En el router principal agrega las rutas del carrito:

```tsx
// src/presentation/router/AppRouter.tsx  (fragmento)

import CartPage from '@/presentation/pages/cart/CartPage'
import ProductDetailPage from '@/presentation/pages/catalog/ProductDetailPage'

// Dentro del elemento de rutas protegidas:
<Route path="catalog/:id" element={<ProductDetailPage />} />
<Route path="cart" element={<CartPage />} />
```

Asegúrate de que `CartPage` esté dentro del guardia de autenticación que redirige a `/login` si no
hay sesión activa.

---

## 6.8 Checkpoint y tabla resumen

Antes de continuar con el módulo 7, verifica que:

1. Agrega un producto desde `/catalog/:id` → el badge del carrito muestra `1`.
2. Recarga la página → el badge sigue mostrando `1` (persistencia en localStorage).
3. En `/cart` puedes incrementar, decrementar y eliminar ítems.
4. Al decrementar a 0 con el botón de basura o desde el store, el ítem desaparece.
5. El botón "Proceder al pago" navega a `/orders/new` (ruta del módulo 7).
6. (Opcional) El `CartDrawer` abre y cierra correctamente desde el header.

### Tabla resumen del módulo 6

| Archivo | Capa | Descripción |
|---|---|---|
| `src/domain/model/cart.types.ts` | Dominio | `CartItem`, `CartState`, `CartSummary` |
| `src/domain/store/cart.store.ts` | Dominio | Store Zustand con `persist` a localStorage |
| `src/presentation/components/CartIcon.tsx` | Presentación | Icono con badge de cantidad |
| `src/presentation/components/CartDrawer.tsx` | Presentación | Panel lateral (Sheet) opcional |
| `src/presentation/pages/catalog/ProductDetailPage.tsx` | Presentación | Detalle de producto completo |
| `src/presentation/pages/cart/CartPage.tsx` | Presentación | Página de carrito con resumen |
