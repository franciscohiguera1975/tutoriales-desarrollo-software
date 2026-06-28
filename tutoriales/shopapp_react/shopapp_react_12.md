# ShopApp React Web — Módulo 12

## Admin Órdenes — Listado, filtros y gestión de estado

**Duración estimada:** 3 horas
**Prerequisitos:** Módulos 1–11 completados. El backend expone `/api/orders/` con soporte para filtros `?status=` y `?search=` y requiere `is_staff=True` para ver todas las órdenes.

---

> **Objetivo**
> Implementar la gestión completa de órdenes desde el panel de administración: extender el API layer, el AdminStore y construir dos páginas — el listado con filtros por estado y la vista detallada — usando componentes reutilizables como `StatusSelect` integrado con shadcn/ui.
>
> **Checkpoint final**
> - La ruta `/admin/orders` lista todas las órdenes del sistema (solo accesible para staff).
> - Los tabs de estado filtran la lista en tiempo real sin recargar la página.
> - El `StatusSelect` inline permite cambiar el estado de una orden directamente desde la tabla.
> - La ruta `/admin/orders/:id` muestra el detalle completo: cliente, ítems y total.
> - `npm run build` compila sin errores de TypeScript.

---

## 12.1 Funciones API de órdenes para admin

**Archivo:** `src/data/api/admin.api.ts` (sección a agregar)

Este módulo centraliza todas las llamadas HTTP del panel de administración. Las funciones de órdenes se agregan al mismo archivo `admin.api.ts` que ya contiene funciones de otros módulos (productos, categorías, etc.), manteniendo un único punto de entrada para las operaciones de staff.

El endpoint `/orders/` usa los query params estándar de DRF: `page` para paginación, `status` para filtrar por estado y `search` para búsqueda por texto. El backend filtra automáticamente por usuario cuando `is_staff=False`, por lo que un staff ve todas las órdenes sin parámetros extra.

```typescript
// src/data/api/admin.api.ts
// --- Agregar después de las funciones existentes ---

import type { Order, OrderStatus } from "@/domain/model/order.types";
import type { PaginatedResponse } from "@/domain/model/catalog.types";
import { apiClient } from "./client"; // instancia Axios con baseURL y headers

// Retorna todas las órdenes del sistema paginadas.
// El backend aplica is_staff check en el ViewSet; si el token no es de staff,
// retorna 403. Los parámetros son opcionales: sin ellos devuelve página 1 sin filtros.
export async function getAdminOrders(
  page = 1,
  status?: OrderStatus | "",
  search?: string
): Promise<PaginatedResponse<Order>> {
  const params: Record<string, string | number> = { page };
  if (status) params.status = status;
  if (search) params.search = search;

  const { data } = await apiClient.get<PaginatedResponse<Order>>("/orders/", {
    params,
  });
  return data;
}

// Obtiene una orden individual con todos sus ítems y datos de usuario.
export async function getAdminOrderById(id: number): Promise<Order> {
  const { data } = await apiClient.get<Order>(`/orders/${id}/`);
  return data;
}

// Actualiza únicamente el campo `status` de la orden mediante PATCH.
// El backend valida que la transición de estado sea legal (ej. no se puede
// pasar de 'delivered' a 'pending') y retorna la orden actualizada.
export async function updateOrderStatus(
  id: number,
  status: OrderStatus
): Promise<Order> {
  const { data } = await apiClient.patch<Order>(`/orders/${id}/`, { status });
  return data;
}
```

El uso de `PATCH` en lugar de `PUT` es importante: solo enviamos el campo que cambia, evitando sobreescribir datos que no conocemos (como los ítems o el total calculado por el servidor).

---

## 12.2 AdminStore — estado de órdenes

**Archivo:** `src/domain/store/admin.store.ts` (sección a agregar)

El AdminStore centraliza todo el estado mutable del panel de administración. Se extiende el store existente agregando la slice de órdenes. Zustand permite componer el estado por secciones sin crear stores separados, lo que facilita acciones cruzadas (ej. actualizar un ítem de la lista después de cambiar su estado).

`ordersFilter` almacena tanto el estado de filtro como la página actual. Cuando el filtro cambia, la acción `setOrdersFilter` resetea la página a 1 automáticamente para evitar mostrar una página inexistente con el nuevo filtro aplicado.

```typescript
// src/domain/store/admin.store.ts
// --- Agregar la siguiente slice al store existente ---

import { create } from "zustand";
import type { Order, OrderStatus } from "@/domain/model/order.types";
import {
  getAdminOrders,
  updateOrderStatus as apiUpdateOrderStatus,
} from "@/data/api/admin.api";

// Tipos de la slice de órdenes
interface OrdersFilter {
  status: OrderStatus | "";
  page: number;
}

interface AdminOrdersSlice {
  adminOrders: Order[];
  ordersLoading: boolean;
  ordersTotal: number;
  ordersFilter: OrdersFilter;

  fetchAdminOrders: () => Promise<void>;
  updateOrderStatus: (id: number, status: OrderStatus) => Promise<void>;
  setOrdersFilter: (partial: Partial<OrdersFilter>) => void;
}

// Si el store ya existe, exportar esta slice e integrarla con immer o combinarla.
// Aquí se muestra como store autónomo para claridad; en el proyecto real
// se fusiona con el store existente usando el patrón de slices de Zustand.
export const useAdminOrdersStore = create<AdminOrdersSlice>((set, get) => ({
  adminOrders: [],
  ordersLoading: false,
  ordersTotal: 0,
  ordersFilter: { status: "", page: 1 },

  // Llama al API con los filtros actuales del store y actualiza el estado.
  // El indicador ordersLoading se activa antes de la llamada y se desactiva
  // tanto en éxito como en error para no bloquear la UI.
  fetchAdminOrders: async () => {
    const { ordersFilter } = get();
    set({ ordersLoading: true });
    try {
      const data = await getAdminOrders(
        ordersFilter.page,
        ordersFilter.status || undefined,
        undefined
      );
      set({ adminOrders: data.results, ordersTotal: data.count });
    } finally {
      set({ ordersLoading: false });
    }
  },

  // Actualiza el estado de una orden en el backend y luego actualiza
  // el objeto en el array local para evitar un re-fetch completo.
  // Esto mantiene la UI reactiva sin una llamada extra de lectura.
  updateOrderStatus: async (id: number, status: OrderStatus) => {
    const updated = await apiUpdateOrderStatus(id, status);
    set((state) => ({
      adminOrders: state.adminOrders.map((o) =>
        o.id === id ? { ...o, status: updated.status } : o
      ),
    }));
  },

  // Al cambiar el filtro se resetea siempre a página 1 para evitar
  // mostrar resultados vacíos cuando el nuevo filtro tiene menos páginas.
  setOrdersFilter: (partial) => {
    set((state) => ({
      ordersFilter: { ...state.ordersFilter, ...partial, page: 1 },
    }));
  },
}));
```

La actualización local tras `updateOrderStatus` es un patrón optimista parcial: no asumimos el nuevo valor antes de que el servidor confirme (hacemos await), pero tampoco refrescamos toda la lista, lo que evita un parpadeo visual innecesario.

---

## 12.3 Componente StatusSelect

**Archivo:** `src/presentation/components/admin/StatusSelect.tsx`

`StatusSelect` es un componente inline que muestra el estado actual de una orden como un selector interactivo. Se usa dentro de la tabla de órdenes para que el administrador cambie el estado sin salir de la lista. Internamente llama a `updateOrderStatus` del store y muestra un spinner mientras la petición está en vuelo.

Los colores de las opciones coinciden con el `StatusBadge` del Módulo 7 para mantener consistencia visual: el administrador ve los mismos colores en la badge de detalle que en el selector de la tabla.

```tsx
// src/presentation/components/admin/StatusSelect.tsx

import { useState } from "react";
import { Loader2 } from "lucide-react";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { useAdminOrdersStore } from "@/domain/store/admin.store";
import type { OrderStatus } from "@/domain/model/order.types";

// Mapa de colores: cada estado tiene un color de texto para diferenciar
// visualmente las opciones del desplegable. Los valores son clases de Tailwind.
const STATUS_COLORS: Record<OrderStatus, string> = {
  pending: "text-yellow-600",
  confirmed: "text-blue-600",
  shipped: "text-purple-600",
  delivered: "text-green-600",
  cancelled: "text-red-600",
};

// Etiquetas en español para mostrar en la UI.
const STATUS_LABELS: Record<OrderStatus, string> = {
  pending: "Pendiente",
  confirmed: "Confirmado",
  shipped: "Enviado",
  delivered: "Entregado",
  cancelled: "Cancelado",
};

const ALL_STATUSES: OrderStatus[] = [
  "pending",
  "confirmed",
  "shipped",
  "delivered",
  "cancelled",
];

interface StatusSelectProps {
  orderId: number;
  currentStatus: OrderStatus;
  // Callback opcional para notificar al padre que el estado cambió.
  onUpdate?: (newStatus: OrderStatus) => void;
}

export function StatusSelect({
  orderId,
  currentStatus,
  onUpdate,
}: StatusSelectProps) {
  const updateOrderStatus = useAdminOrdersStore((s) => s.updateOrderStatus);
  const [updating, setUpdating] = useState(false);

  const handleChange = async (newStatus: string) => {
    if (newStatus === currentStatus) return;
    setUpdating(true);
    try {
      await updateOrderStatus(orderId, newStatus as OrderStatus);
      onUpdate?.(newStatus as OrderStatus);
    } finally {
      setUpdating(false);
    }
  };

  // Mientras actualiza, se muestra un spinner en lugar del selector
  // para dejar claro que hay una operación en curso y evitar doble click.
  if (updating) {
    return (
      <div className="flex items-center gap-1 text-muted-foreground text-sm">
        <Loader2 className="h-4 w-4 animate-spin" />
        <span>Actualizando…</span>
      </div>
    );
  }

  return (
    <Select value={currentStatus} onValueChange={handleChange}>
      <SelectTrigger className="w-36 h-8 text-sm">
        <SelectValue />
      </SelectTrigger>
      <SelectContent>
        {ALL_STATUSES.map((status) => (
          <SelectItem key={status} value={status}>
            <span className={STATUS_COLORS[status]}>
              {STATUS_LABELS[status]}
            </span>
          </SelectItem>
        ))}
      </SelectContent>
    </Select>
  );
}
```

El componente es controlado: recibe `currentStatus` como prop y el valor real proviene del store. Cuando el store actualiza el array `adminOrders`, React re-renderiza la fila con el nuevo `currentStatus`, manteniendo la fuente de verdad en un único lugar.

---

## 12.4 Página AdminOrdersPage

**Archivo:** `src/presentation/pages/admin/AdminOrdersPage.tsx`

Esta página es el corazón de la gestión de órdenes. Combina tabs de filtro por estado, una tabla paginada con datos reales y el componente `StatusSelect` inline. Está envuelta en `AdminShell` para heredar el layout de navegación lateral del panel.

Los tabs muestran el total de órdenes por estado en un badge. Para esto el store expone `ordersTotal` que refleja el `count` de la respuesta paginada con el filtro activo. Si se necesitan conteos por cada estado simultáneamente, se requeriría una llamada adicional al backend con un endpoint de estadísticas (fuera del alcance de este módulo).

```tsx
// src/presentation/pages/admin/AdminOrdersPage.tsx

import { useEffect } from "react";
import { Link } from "react-router-dom";
import { Eye } from "lucide-react";
import { AdminShell } from "@/presentation/components/AdminShell";
import { StatusSelect } from "@/presentation/components/admin/StatusSelect";
import { useAdminOrdersStore } from "@/domain/store/admin.store";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import { Tabs, TabsList, TabsTrigger } from "@/components/ui/tabs";
import type { OrderStatus } from "@/domain/model/order.types";

// Configuración de tabs: valor vacío representa "Todas".
const STATUS_TABS: { label: string; value: OrderStatus | "" }[] = [
  { label: "Todas", value: "" },
  { label: "Pendiente", value: "pending" },
  { label: "Confirmado", value: "confirmed" },
  { label: "Enviado", value: "shipped" },
  { label: "Entregado", value: "delivered" },
  { label: "Cancelado", value: "cancelled" },
];

const PAGE_SIZE = 10;

export function AdminOrdersPage() {
  const {
    adminOrders,
    ordersLoading,
    ordersTotal,
    ordersFilter,
    fetchAdminOrders,
    setOrdersFilter,
  } = useAdminOrdersStore();

  // Cada vez que cambia el filtro (estado o página), se vuelve a buscar.
  useEffect(() => {
    fetchAdminOrders();
  }, [ordersFilter, fetchAdminOrders]);

  const totalPages = Math.ceil(ordersTotal / PAGE_SIZE);

  const handleTabChange = (value: string) => {
    setOrdersFilter({ status: value as OrderStatus | "" });
  };

  const handlePageChange = (newPage: number) => {
    setOrdersFilter({ page: newPage });
  };

  // Filas skeleton: se muestran durante la carga para evitar el salto visual.
  const skeletonRows = Array.from({ length: 6 });

  return (
    <AdminShell>
      <div className="space-y-6">
        {/* Cabecera */}
        <div className="flex items-center justify-between">
          <div>
            <h1 className="text-2xl font-semibold tracking-tight">Órdenes</h1>
            <p className="text-sm text-muted-foreground">
              {ordersTotal} {ordersTotal === 1 ? "orden" : "órdenes"} en total
            </p>
          </div>
        </div>

        {/* Tabs de filtro por estado */}
        <Tabs value={ordersFilter.status} onValueChange={handleTabChange}>
          <TabsList className="flex-wrap h-auto gap-1">
            {STATUS_TABS.map((tab) => (
              <TabsTrigger
                key={tab.value}
                value={tab.value}
                className="text-xs"
              >
                {tab.label}
                {tab.value === ordersFilter.status && ordersTotal > 0 && (
                  <Badge
                    variant="secondary"
                    className="ml-1.5 h-4 px-1 text-xs"
                  >
                    {ordersTotal}
                  </Badge>
                )}
              </TabsTrigger>
            ))}
          </TabsList>
        </Tabs>

        {/* Tabla de órdenes */}
        <div className="rounded-md border">
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead className="w-24"># Orden</TableHead>
                <TableHead>Cliente</TableHead>
                <TableHead>Fecha</TableHead>
                <TableHead className="text-center">Ítems</TableHead>
                <TableHead className="text-right">Total</TableHead>
                <TableHead className="w-44">Estado</TableHead>
                <TableHead className="w-12 text-center">Ver</TableHead>
              </TableRow>
            </TableHeader>
            <TableBody>
              {ordersLoading
                ? // Skeleton rows durante la carga
                  skeletonRows.map((_, i) => (
                    <TableRow key={i}>
                      {Array.from({ length: 7 }).map((__, j) => (
                        <TableCell key={j}>
                          <Skeleton className="h-5 w-full" />
                        </TableCell>
                      ))}
                    </TableRow>
                  ))
                : adminOrders.length === 0
                ? // Estado vacío
                  (
                    <TableRow>
                      <TableCell
                        colSpan={7}
                        className="h-32 text-center text-muted-foreground"
                      >
                        No se encontraron órdenes con el filtro seleccionado.
                      </TableCell>
                    </TableRow>
                  )
                : adminOrders.map((order) => (
                    <TableRow
                      key={order.id}
                      className="cursor-pointer hover:bg-muted/50"
                    >
                      <TableCell className="font-mono text-sm font-medium">
                        #{order.id}
                      </TableCell>
                      <TableCell className="font-medium">
                        {order.user?.username ?? "—"}
                      </TableCell>
                      <TableCell className="text-sm text-muted-foreground">
                        {new Date(order.created_at).toLocaleDateString(
                          "es-EC",
                          {
                            year: "numeric",
                            month: "short",
                            day: "numeric",
                          }
                        )}
                      </TableCell>
                      <TableCell className="text-center">
                        <Badge variant="outline">{order.total_items}</Badge>
                      </TableCell>
                      <TableCell className="text-right font-medium">
                        ${Number(order.total).toFixed(2)}
                      </TableCell>
                      <TableCell
                        // Detener propagación para que el click en el select
                        // no navegue a la página de detalle.
                        onClick={(e) => e.stopPropagation()}
                      >
                        <StatusSelect
                          orderId={order.id}
                          currentStatus={order.status}
                        />
                      </TableCell>
                      <TableCell className="text-center">
                        <Link
                          to={`/admin/orders/${order.id}`}
                          onClick={(e) => e.stopPropagation()}
                        >
                          <Button variant="ghost" size="icon" className="h-8 w-8">
                            <Eye className="h-4 w-4" />
                            <span className="sr-only">
                              Ver detalle orden #{order.id}
                            </span>
                          </Button>
                        </Link>
                      </TableCell>
                    </TableRow>
                  ))}
            </TableBody>
          </Table>
        </div>

        {/* Controles de paginación */}
        {totalPages > 1 && (
          <div className="flex items-center justify-between text-sm text-muted-foreground">
            <span>
              Página {ordersFilter.page} de {totalPages}
            </span>
            <div className="flex gap-2">
              <Button
                variant="outline"
                size="sm"
                disabled={ordersFilter.page <= 1}
                onClick={() => handlePageChange(ordersFilter.page - 1)}
              >
                Anterior
              </Button>
              <Button
                variant="outline"
                size="sm"
                disabled={ordersFilter.page >= totalPages}
                onClick={() => handlePageChange(ordersFilter.page + 1)}
              >
                Siguiente
              </Button>
            </div>
          </div>
        )}
      </div>
    </AdminShell>
  );
}
```

El `onClick={(e) => e.stopPropagation()}` en la celda de Estado y en el botón Ver es necesario porque toda la fila tiene `cursor-pointer`; sin esto, hacer clic en el select o en el ícono también dispararía la navegación al detalle.

---

## 12.5 Página AdminOrderDetailPage

**Archivo:** `src/presentation/pages/admin/AdminOrderDetailPage.tsx`

La página de detalle muestra la información completa de una orden: cabecera con número y fecha, estado editable mediante `StatusSelect`, información del cliente, tabla de ítems con precios y subtotales, y el total general. Se accede desde `/admin/orders/:id`.

```tsx
// src/presentation/pages/admin/AdminOrderDetailPage.tsx

import { useEffect, useState } from "react";
import { useParams, useNavigate } from "react-router-dom";
import { ArrowLeft, Package } from "lucide-react";
import { AdminShell } from "@/presentation/components/AdminShell";
import { StatusSelect } from "@/presentation/components/admin/StatusSelect";
import { getAdminOrderById } from "@/data/api/admin.api";
import { Button } from "@/components/ui/button";
import { Skeleton } from "@/components/ui/skeleton";
import { Separator } from "@/components/ui/separator";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import type { Order } from "@/domain/model/order.types";

export function AdminOrderDetailPage() {
  const { id } = useParams<{ id: string }>();
  const navigate = useNavigate();
  const [order, setOrder] = useState<Order | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!id) return;
    setLoading(true);
    setError(null);
    getAdminOrderById(Number(id))
      .then(setOrder)
      .catch(() => setError("No se pudo cargar la orden. Intenta de nuevo."))
      .finally(() => setLoading(false));
  }, [id]);

  // Cuando StatusSelect actualiza el estado en el store, también actualizamos
  // el objeto local para que la cabecera refleje el nuevo estado sin recargar.
  const handleStatusUpdate = (newStatus: Order["status"]) => {
    if (order) setOrder({ ...order, status: newStatus });
  };

  return (
    <AdminShell>
      {/* Botón volver */}
      <Button
        variant="ghost"
        size="sm"
        className="mb-6 -ml-2"
        onClick={() => navigate("/admin/orders")}
      >
        <ArrowLeft className="mr-2 h-4 w-4" />
        Volver a órdenes
      </Button>

      {loading && (
        <div className="space-y-4">
          <Skeleton className="h-8 w-48" />
          <Skeleton className="h-4 w-72" />
          <Skeleton className="h-64 w-full" />
        </div>
      )}

      {error && (
        <div className="rounded-md bg-destructive/10 p-4 text-destructive text-sm">
          {error}
        </div>
      )}

      {!loading && !error && order && (
        <div className="space-y-8">
          {/* Cabecera de la orden */}
          <div className="flex flex-col gap-4 sm:flex-row sm:items-start sm:justify-between">
            <div>
              <div className="flex items-center gap-3">
                <Package className="h-5 w-5 text-muted-foreground" />
                <h1 className="text-2xl font-semibold tracking-tight">
                  Orden #{order.id}
                </h1>
              </div>
              <p className="mt-1 text-sm text-muted-foreground">
                Creada el{" "}
                {new Date(order.created_at).toLocaleDateString("es-EC", {
                  weekday: "long",
                  year: "numeric",
                  month: "long",
                  day: "numeric",
                })}
              </p>
            </div>
            {/* Estado editable directamente en el detalle */}
            <div className="flex items-center gap-2">
              <span className="text-sm text-muted-foreground">Estado:</span>
              <StatusSelect
                orderId={order.id}
                currentStatus={order.status}
                onUpdate={handleStatusUpdate}
              />
            </div>
          </div>

          <Separator />

          {/* Información del cliente */}
          <section>
            <h2 className="mb-3 text-base font-semibold">Cliente</h2>
            <dl className="grid grid-cols-2 gap-x-4 gap-y-2 text-sm sm:grid-cols-3">
              <div>
                <dt className="text-muted-foreground">Usuario</dt>
                <dd className="font-medium">{order.user?.username ?? "—"}</dd>
              </div>
              <div>
                <dt className="text-muted-foreground">Email</dt>
                <dd className="font-medium">{order.user?.email ?? "—"}</dd>
              </div>
              <div>
                <dt className="text-muted-foreground">Fecha de orden</dt>
                <dd className="font-medium">
                  {new Date(order.created_at).toLocaleDateString("es-EC")}
                </dd>
              </div>
            </dl>
          </section>

          <Separator />

          {/* Tabla de ítems */}
          <section>
            <h2 className="mb-3 text-base font-semibold">
              Ítems ({order.total_items})
            </h2>
            <div className="rounded-md border">
              <Table>
                <TableHeader>
                  <TableRow>
                    <TableHead>Producto</TableHead>
                    <TableHead className="text-center w-24">Cantidad</TableHead>
                    <TableHead className="text-right w-32">
                      Precio unit.
                    </TableHead>
                    <TableHead className="text-right w-32">Subtotal</TableHead>
                  </TableRow>
                </TableHeader>
                <TableBody>
                  {order.items.map((item) => (
                    <TableRow key={item.id}>
                      <TableCell className="font-medium">
                        {item.product_name ?? `Producto #${item.product}`}
                      </TableCell>
                      <TableCell className="text-center">
                        {item.quantity}
                      </TableCell>
                      <TableCell className="text-right">
                        ${Number(item.unit_price).toFixed(2)}
                      </TableCell>
                      <TableCell className="text-right">
                        $
                        {(
                          Number(item.unit_price) * item.quantity
                        ).toFixed(2)}
                      </TableCell>
                    </TableRow>
                  ))}
                </TableBody>
              </Table>
            </div>

            {/* Total */}
            <div className="mt-4 flex justify-end">
              <div className="w-64 space-y-1 text-sm">
                <div className="flex justify-between text-muted-foreground">
                  <span>Subtotal ({order.total_items} ítems)</span>
                  <span>${Number(order.total).toFixed(2)}</span>
                </div>
                <Separator />
                <div className="flex justify-between text-base font-semibold">
                  <span>Total</span>
                  <span>${Number(order.total).toFixed(2)}</span>
                </div>
              </div>
            </div>
          </section>
        </div>
      )}
    </AdminShell>
  );
}
```

**Registro de la ruta en el router:**

```tsx
// src/presentation/router/AppRouter.tsx
// --- Agregar dentro del bloque de rutas protegidas de staff ---

import { AdminOrdersPage } from "@/presentation/pages/admin/AdminOrdersPage";
import { AdminOrderDetailPage } from "@/presentation/pages/admin/AdminOrderDetailPage";

// Dentro del elemento <Route> de rutas admin:
<Route path="/admin/orders" element={<AdminOrdersPage />} />
<Route path="/admin/orders/:id" element={<AdminOrderDetailPage />} />
```

El componente obtiene el `id` desde `useParams` y hace su propia llamada al API en lugar de leer del store. Esto es correcto para páginas de detalle: el store de lista no garantiza que el ítem esté cargado (el usuario puede navegar directamente por URL), y la llamada individual incluye los ítems completos que la lista paginada puede omitir por rendimiento.

---

## 12.6 Checkpoint y tabla resumen

Antes de continuar con el Módulo 13, verifica que cada punto funcione de forma independiente:

1. Navega a `/admin/orders` — la tabla debe cargar con las órdenes reales del backend.
2. Cambia el tab a "Pendiente" — la tabla debe actualizarse sin recargar la página.
3. En cualquier fila, cambia el estado con el `StatusSelect` — debe mostrar el spinner brevemente y luego actualizar el valor en la misma fila.
4. Haz clic en el ícono de ojo (Ver) de cualquier orden — debe navegar a `/admin/orders/{id}`.
5. En la página de detalle, cambia el estado — la cabecera debe reflejar el nuevo valor.
6. Ejecuta `npm run build` y verifica que no haya errores de TypeScript.

### Archivos creados o modificados en este módulo

| Archivo | Tipo | Descripción |
|---|---|---|
| `src/data/api/admin.api.ts` | Modificado | Funciones `getAdminOrders`, `getAdminOrderById`, `updateOrderStatus` |
| `src/domain/store/admin.store.ts` | Modificado | Slice de órdenes: estado, `fetchAdminOrders`, `updateOrderStatus`, `setOrdersFilter` |
| `src/presentation/components/admin/StatusSelect.tsx` | Nuevo | Selector inline de estado con spinner y colores por estado |
| `src/presentation/pages/admin/AdminOrdersPage.tsx` | Nuevo | Listado de órdenes con tabs de filtro, tabla paginada y `StatusSelect` inline |
| `src/presentation/pages/admin/AdminOrderDetailPage.tsx` | Nuevo | Detalle completo de orden: cliente, ítems, total y estado editable |
| `src/presentation/router/AppRouter.tsx` | Modificado | Rutas `/admin/orders` y `/admin/orders/:id` |
