# ShopApp React Web — Módulo 11

## Admin Productos — CRUD completo con formulario y restock

**Duración estimada:** 4 horas
**Prerequisitos:** Módulos 1–10

---

> **Objetivo**
> Construir el panel de administración de productos con CRUD completo, búsqueda con debounce,
> paginación, y una operación especial de restock. El formulario de producto es más complejo
> que el de categorías: maneja precio, stock, categoría (select) y estado activo/inactivo
> (switch). Se reutiliza `DeleteConfirmDialog` del Módulo 10 sin modificaciones.
>
> **Checkpoint final**
> - La ruta `/admin/products` muestra una tabla paginada de todos los productos (incluidos los inactivos).
> - El buscador filtra por nombre con un debounce de 400 ms sin recargar la página.
> - "Nuevo producto" abre un diálogo con formulario completo validado con Zod.
> - El ícono de edición abre el formulario pre-poblado con los datos del producto.
> - El botón de restock permite añadir unidades al stock actual.
> - El switch de estado activo/inactivo en la tabla funciona inline.
> - El ícono de eliminar abre el AlertDialog de confirmación reutilizado del módulo 10.

---

## 11.1 Funciones API de admin para productos

Se extiende `src/data/api/admin.api.ts` con las funciones CRUD de productos. El endpoint
`getAdminProducts` acepta parámetros opcionales de paginación y búsqueda, y el staff ve
todos los productos independientemente de `is_active`.

```typescript
// src/data/api/admin.api.ts  (se agrega a continuación de lo existente del módulo 10)
import apiClient from "@/data/api/client";
import type { Category, Product } from "@/domain/model/catalog.types";

// ── Tipos ya definidos en módulo 10 ───────────────────────────────────────────
// CategoryPayload, ProductPayload (actualizamos ProductPayload abajo)

export interface ProductPayload {
  name: string;
  description?: string;
  price: number;
  stock: number;
  category: number;
  is_active: boolean;
}

export interface PaginatedProducts {
  count: number;
  next: string | null;
  previous: string | null;
  results: Product[];
}

// ── Productos ─────────────────────────────────────────────────────────────────

/**
 * GET /products/?page=&search=
 * El usuario staff recibe todos los productos, incluyendo los inactivos.
 */
export async function getAdminProducts(
  page = 1,
  search = ""
): Promise<PaginatedProducts> {
  const params: Record<string, string | number> = { page };
  if (search.trim()) {
    params.search = search.trim();
  }
  const response = await apiClient.get<PaginatedProducts>("/products/", {
    params,
  });
  return response.data;
}

/** POST /products/ — crea un nuevo producto */
export async function createProduct(data: ProductPayload): Promise<Product> {
  const response = await apiClient.post<Product>("/products/", data);
  return response.data;
}

/** PATCH /products/{id}/ — actualiza parcialmente un producto */
export async function updateProduct(
  id: number,
  data: Partial<ProductPayload>
): Promise<Product> {
  const response = await apiClient.patch<Product>(`/products/${id}/`, data);
  return response.data;
}

/** DELETE /products/{id}/ — elimina un producto */
export async function deleteProduct(id: number): Promise<void> {
  await apiClient.delete(`/products/${id}/`);
}

/**
 * PATCH /products/{id}/ — incrementa el stock sumando `quantity` al valor actual.
 * Recibe el producto completo para conocer el stock actual sin necesidad de
 * hacer una petición GET adicional.
 */
export async function restockProduct(
  id: number,
  currentStock: number,
  quantity: number
): Promise<Product> {
  const response = await apiClient.patch<Product>(`/products/${id}/`, {
    stock: currentStock + quantity,
  });
  return response.data;
}
```

> **Nota sobre paginación:** La respuesta de Django REST Framework con `PageNumberPagination`
> tiene la forma `{ count, next, previous, results }`. El tipo `PaginatedProducts` modela
> esta estructura. Asegúrate de que el backend tenga `PAGE_SIZE` configurado en `settings.py`.

---

## 11.2 AdminStore — estado y acciones de productos

Se extiende el store de admin con un slice completo para productos que maneja paginación,
búsqueda y las cinco acciones CRUD incluyendo restock.

```typescript
// src/domain/store/admin.store.ts  (versión completa con módulo 10 + módulo 11)
import { create } from "zustand";
import type { Category, Product } from "@/domain/model/catalog.types";
import {
  getCategories,
  createCategory as apiCreateCategory,
  updateCategory as apiUpdateCategory,
  deleteCategory as apiDeleteCategory,
  getAdminProducts,
  createProduct as apiCreateProduct,
  updateProduct as apiUpdateProduct,
  deleteProduct as apiDeleteProduct,
  restockProduct as apiRestockProduct,
  type CategoryPayload,
  type ProductPayload,
} from "@/data/api/admin.api";

// ── Estado ────────────────────────────────────────────────────────────────────

interface AdminState {
  // ── Categorías (módulo 10) ─────────────────────────────────────────────────
  categories: Category[];
  categoriesLoading: boolean;
  categoriesError: string | null;
  fetchAdminCategories: () => Promise<void>;
  createCategory: (data: CategoryPayload) => Promise<void>;
  updateCategory: (id: number, data: Partial<CategoryPayload>) => Promise<void>;
  deleteCategory: (id: number) => Promise<void>;

  // ── Productos (módulo 11) ──────────────────────────────────────────────────
  adminProducts: Product[];
  productsLoading: boolean;
  productsError: string | null;
  productsTotal: number;
  fetchAdminProducts: (page?: number, search?: string) => Promise<void>;
  createProduct: (data: ProductPayload) => Promise<void>;
  updateProduct: (id: number, data: Partial<ProductPayload>) => Promise<void>;
  deleteProduct: (id: number) => Promise<void>;
  restockProduct: (id: number, qty: number) => Promise<void>;
}

// ── Store ─────────────────────────────────────────────────────────────────────

export const useAdminStore = create<AdminState>((set, get) => ({
  // ── Estado inicial — categorías ──────────────────────────────────────────
  categories: [],
  categoriesLoading: false,
  categoriesError: null,

  fetchAdminCategories: async () => {
    set({ categoriesLoading: true, categoriesError: null });
    try {
      const categories = await getCategories();
      set({ categories, categoriesLoading: false });
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "Error al cargar categorías";
      set({ categoriesError: message, categoriesLoading: false });
    }
  },

  createCategory: async (data) => {
    const newCategory = await apiCreateCategory(data);
    set((state) => ({ categories: [...state.categories, newCategory] }));
  },

  updateCategory: async (id, data) => {
    const updated = await apiUpdateCategory(id, data);
    set((state) => ({
      categories: state.categories.map((c) => (c.id === id ? updated : c)),
    }));
  },

  deleteCategory: async (id) => {
    await apiDeleteCategory(id);
    set((state) => ({
      categories: state.categories.filter((c) => c.id !== id),
    }));
  },

  // ── Estado inicial — productos ───────────────────────────────────────────
  adminProducts: [],
  productsLoading: false,
  productsError: null,
  productsTotal: 0,

  // ── fetchAdminProducts ───────────────────────────────────────────────────
  fetchAdminProducts: async (page = 1, search = "") => {
    set({ productsLoading: true, productsError: null });
    try {
      const data = await getAdminProducts(page, search);
      set({
        adminProducts: data.results,
        productsTotal: data.count,
        productsLoading: false,
      });
    } catch (error) {
      const message =
        error instanceof Error ? error.message : "Error al cargar productos";
      set({ productsError: message, productsLoading: false });
    }
  },

  // ── createProduct ────────────────────────────────────────────────────────
  createProduct: async (data) => {
    const newProduct = await apiCreateProduct(data);
    set((state) => ({
      adminProducts: [newProduct, ...state.adminProducts],
      productsTotal: state.productsTotal + 1,
    }));
  },

  // ── updateProduct ────────────────────────────────────────────────────────
  updateProduct: async (id, data) => {
    const updated = await apiUpdateProduct(id, data);
    set((state) => ({
      adminProducts: state.adminProducts.map((p) =>
        p.id === id ? updated : p
      ),
    }));
  },

  // ── deleteProduct ────────────────────────────────────────────────────────
  deleteProduct: async (id) => {
    await apiDeleteProduct(id);
    set((state) => ({
      adminProducts: state.adminProducts.filter((p) => p.id !== id),
      productsTotal: state.productsTotal - 1,
    }));
  },

  // ── restockProduct ───────────────────────────────────────────────────────
  restockProduct: async (id, qty) => {
    // Obtenemos el stock actual del producto en el estado local
    const product = get().adminProducts.find((p) => p.id === id);
    if (!product) throw new Error(`Producto ${id} no encontrado en el estado`);
    const updated = await apiRestockProduct(id, product.stock, qty);
    set((state) => ({
      adminProducts: state.adminProducts.map((p) =>
        p.id === id ? updated : p
      ),
    }));
  },
}));
```

---

## 11.3 Componente ProductForm

`ProductForm` es el formulario más completo del panel admin. Gestiona cinco campos con tipos
distintos: texto, número, select (categoría), y switch (estado activo). El esquema Zod usa
`z.coerce.number()` para manejar la conversión automática de los valores de los inputs HTML
que siempre devuelven strings.

```tsx
// src/presentation/components/admin/ProductForm.tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import {
  Form,
  FormControl,
  FormDescription,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Button } from "@/components/ui/button";
import { Switch } from "@/components/ui/switch";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import { Loader2 } from "lucide-react";
import type { Category } from "@/domain/model/catalog.types";

// ── Schema Zod ────────────────────────────────────────────────────────────────

const productSchema = z.object({
  name: z
    .string()
    .min(2, "El nombre debe tener al menos 2 caracteres")
    .max(200, "El nombre no puede superar los 200 caracteres"),
  description: z
    .string()
    .max(1000, "La descripción no puede superar los 1000 caracteres")
    .optional(),
  price: z.coerce
    .number({ invalid_type_error: "El precio debe ser un número" })
    .positive("El precio debe ser mayor a 0"),
  stock: z.coerce
    .number({ invalid_type_error: "El stock debe ser un número" })
    .int("El stock debe ser un número entero")
    .min(0, "El stock no puede ser negativo"),
  category: z.coerce
    .number({ invalid_type_error: "Selecciona una categoría" })
    .positive("Selecciona una categoría"),
  is_active: z.boolean().default(true),
});

export type ProductFormValues = z.infer<typeof productSchema>;

// ── Props ─────────────────────────────────────────────────────────────────────

interface ProductFormProps {
  defaultValues?: Partial<ProductFormValues>;
  onSubmit: (data: ProductFormValues) => Promise<void>;
  isLoading?: boolean;
  categories: Category[];
}

// ── Componente ────────────────────────────────────────────────────────────────

export function ProductForm({
  defaultValues,
  onSubmit,
  isLoading = false,
  categories,
}: ProductFormProps) {
  const form = useForm<ProductFormValues>({
    resolver: zodResolver(productSchema),
    defaultValues: {
      name: defaultValues?.name ?? "",
      description: defaultValues?.description ?? "",
      price: defaultValues?.price ?? 0,
      stock: defaultValues?.stock ?? 0,
      category: defaultValues?.category ?? 0,
      is_active: defaultValues?.is_active ?? true,
    },
  });

  const handleSubmit = form.handleSubmit(async (data) => {
    await onSubmit(data);
  });

  return (
    <Form {...form}>
      <form onSubmit={handleSubmit} className="space-y-4">
        {/* Nombre */}
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Nombre</FormLabel>
              <FormControl>
                <Input
                  placeholder="Ej. Laptop Gaming 15\""
                  disabled={isLoading}
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Descripción */}
        <FormField
          control={form.control}
          name="description"
          render={({ field }) => (
            <FormItem>
              <FormLabel>
                Descripción{" "}
                <span className="text-muted-foreground text-xs">(opcional)</span>
              </FormLabel>
              <FormControl>
                <Textarea
                  placeholder="Describe el producto..."
                  rows={3}
                  disabled={isLoading}
                  {...field}
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Precio y Stock — en la misma fila */}
        <div className="grid grid-cols-2 gap-4">
          <FormField
            control={form.control}
            name="price"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Precio ($)</FormLabel>
                <FormControl>
                  <Input
                    type="number"
                    step="0.01"
                    min="0"
                    placeholder="0.00"
                    disabled={isLoading}
                    {...field}
                  />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />

          <FormField
            control={form.control}
            name="stock"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Stock</FormLabel>
                <FormControl>
                  <Input
                    type="number"
                    min="0"
                    step="1"
                    placeholder="0"
                    disabled={isLoading}
                    {...field}
                  />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
        </div>

        {/* Categoría */}
        <FormField
          control={form.control}
          name="category"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Categoría</FormLabel>
              <Select
                onValueChange={field.onChange}
                defaultValue={
                  field.value ? String(field.value) : undefined
                }
                disabled={isLoading}
              >
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Selecciona una categoría" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  {categories.map((cat) => (
                    <SelectItem key={cat.id} value={String(cat.id)}>
                      {cat.name}
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        {/* Estado activo */}
        <FormField
          control={form.control}
          name="is_active"
          render={({ field }) => (
            <FormItem className="flex items-center justify-between rounded-lg border p-3">
              <div>
                <FormLabel>Producto activo</FormLabel>
                <FormDescription className="text-xs">
                  Los productos inactivos no son visibles para los clientes.
                </FormDescription>
              </div>
              <FormControl>
                <Switch
                  checked={field.value}
                  onCheckedChange={field.onChange}
                  disabled={isLoading}
                />
              </FormControl>
            </FormItem>
          )}
        />

        {/* Submit */}
        <div className="flex justify-end pt-2">
          <Button type="submit" disabled={isLoading}>
            {isLoading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
            {isLoading ? "Guardando..." : "Guardar"}
          </Button>
        </div>
      </form>
    </Form>
  );
}
```

> **Por qué `z.coerce.number()`:** Los inputs HTML de tipo `number` retornan strings vacíos
> cuando el campo está limpio. `z.coerce` convierte el string al tipo esperado antes de validar,
> evitando errores de tipo confusos. Para el campo `category`, convierte el string del `value`
> del `SelectItem` a número automáticamente.

---

## 11.4 Componente ProductDialog

`ProductDialog` envuelve `ProductForm` en un diálogo de mayor tamaño (`max-w-2xl`) para
acomodar todos los campos del producto. Recibe la lista de categorías como prop para
pasarla al formulario, y determina modo creación o edición según si llega una prop `product`.

```tsx
// src/presentation/components/admin/ProductDialog.tsx
import { useState } from "react";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import { useToast } from "@/components/ui/use-toast";
import type { Category, Product } from "@/domain/model/catalog.types";
import { useAdminStore } from "@/domain/store/admin.store";
import { ProductForm, type ProductFormValues } from "./ProductForm";

// ── Props ─────────────────────────────────────────────────────────────────────

interface ProductDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  product?: Product;
  categories: Category[];
  onSuccess?: () => void;
}

// ── Componente ────────────────────────────────────────────────────────────────

export function ProductDialog({
  open,
  onOpenChange,
  product,
  categories,
  onSuccess,
}: ProductDialogProps) {
  const [isLoading, setIsLoading] = useState(false);
  const { toast } = useToast();
  const createProduct = useAdminStore((s) => s.createProduct);
  const updateProduct = useAdminStore((s) => s.updateProduct);

  const isEditing = Boolean(product);
  const title = isEditing ? "Editar producto" : "Nuevo producto";

  const handleSubmit = async (data: ProductFormValues) => {
    setIsLoading(true);
    try {
      if (isEditing && product) {
        await updateProduct(product.id, data);
        toast({
          title: "Producto actualizado",
          description: `"${data.name}" fue actualizado correctamente.`,
        });
      } else {
        await createProduct(data);
        toast({
          title: "Producto creado",
          description: `"${data.name}" fue creado correctamente.`,
        });
      }
      onOpenChange(false);
      onSuccess?.();
    } catch {
      toast({
        variant: "destructive",
        title: "Error",
        description: isEditing
          ? "No se pudo actualizar el producto."
          : "No se pudo crear el producto.",
      });
    } finally {
      setIsLoading(false);
    }
  };

  // Construimos defaultValues a partir del producto existente
  const defaultValues: Partial<ProductFormValues> | undefined = product
    ? {
        name: product.name,
        description: product.description,
        price: product.price,
        stock: product.stock,
        category: product.category,
        is_active: product.is_active,
      }
    : undefined;

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-2xl max-h-[90vh] overflow-y-auto">
        <DialogHeader>
          <DialogTitle>{title}</DialogTitle>
        </DialogHeader>
        <ProductForm
          defaultValues={defaultValues}
          onSubmit={handleSubmit}
          isLoading={isLoading}
          categories={categories}
        />
      </DialogContent>
    </Dialog>
  );
}
```

> **`max-h-[90vh] overflow-y-auto`:** El formulario de producto es más largo que el de categorías.
> Limitamos la altura del diálogo al 90% del viewport y habilitamos scroll interno para que
> sea usable en pantallas pequeñas sin que el diálogo se salga de la vista.

---

## 11.5 Componente RestockDialog

`RestockDialog` es un diálogo pequeño y enfocado para añadir unidades al stock de un producto.
Muestra el stock actual como referencia y valida que la cantidad ingresada sea un entero
positivo antes de llamar a la acción `restockProduct` del store.

```tsx
// src/presentation/components/admin/RestockDialog.tsx
import { useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogDescription,
} from "@/components/ui/dialog";
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";
import { Loader2, Package } from "lucide-react";
import { useToast } from "@/components/ui/use-toast";
import { useAdminStore } from "@/domain/store/admin.store";
import type { Product } from "@/domain/model/catalog.types";

// ── Schema ────────────────────────────────────────────────────────────────────

const restockSchema = z.object({
  quantity: z.coerce
    .number({ invalid_type_error: "La cantidad debe ser un número" })
    .int("La cantidad debe ser un número entero")
    .positive("La cantidad debe ser mayor a 0"),
});

type RestockFormValues = z.infer<typeof restockSchema>;

// ── Props ─────────────────────────────────────────────────────────────────────

interface RestockDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  product: Product;
  onSuccess?: () => void;
}

// ── Componente ────────────────────────────────────────────────────────────────

export function RestockDialog({
  open,
  onOpenChange,
  product,
  onSuccess,
}: RestockDialogProps) {
  const [isLoading, setIsLoading] = useState(false);
  const { toast } = useToast();
  const restockProduct = useAdminStore((s) => s.restockProduct);

  const form = useForm<RestockFormValues>({
    resolver: zodResolver(restockSchema),
    defaultValues: { quantity: 1 },
  });

  const handleSubmit = form.handleSubmit(async ({ quantity }) => {
    setIsLoading(true);
    try {
      await restockProduct(product.id, quantity);
      toast({
        title: "Stock actualizado",
        description: `Se agregaron ${quantity} unidades a "${product.name}". Nuevo stock: ${product.stock + quantity}.`,
      });
      form.reset();
      onOpenChange(false);
      onSuccess?.();
    } catch {
      toast({
        variant: "destructive",
        title: "Error",
        description: "No se pudo actualizar el stock.",
      });
    } finally {
      setIsLoading(false);
    }
  });

  return (
    <Dialog
      open={open}
      onOpenChange={(isOpen) => {
        if (!isOpen) form.reset();
        onOpenChange(isOpen);
      }}
    >
      <DialogContent className="sm:max-w-sm">
        <DialogHeader>
          <DialogTitle className="flex items-center gap-2">
            <Package className="h-5 w-5" />
            Reabastecer stock
          </DialogTitle>
          <DialogDescription className="text-sm">
            {product.name}
          </DialogDescription>
        </DialogHeader>

        {/* Stock actual */}
        <div className="flex items-center justify-between rounded-lg border p-3 bg-muted/30">
          <span className="text-sm text-muted-foreground">Stock actual</span>
          <Badge variant={product.stock === 0 ? "destructive" : "secondary"}>
            {product.stock} unidades
          </Badge>
        </div>

        {/* Formulario */}
        <Form {...form}>
          <form onSubmit={handleSubmit} className="space-y-4">
            <FormField
              control={form.control}
              name="quantity"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Cantidad a agregar</FormLabel>
                  <FormControl>
                    <Input
                      type="number"
                      min="1"
                      step="1"
                      placeholder="Ej. 50"
                      disabled={isLoading}
                      {...field}
                    />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />

            <div className="flex justify-end gap-2">
              <Button
                type="button"
                variant="outline"
                onClick={() => {
                  form.reset();
                  onOpenChange(false);
                }}
                disabled={isLoading}
              >
                Cancelar
              </Button>
              <Button type="submit" disabled={isLoading}>
                {isLoading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
                {isLoading ? "Guardando..." : "Agregar al stock"}
              </Button>
            </div>
          </form>
        </Form>
      </DialogContent>
    </Dialog>
  );
}
```

---

## 11.6 Página AdminProductsPage

`AdminProductsPage` es la página más completa del panel admin. Integra búsqueda con debounce,
paginación, toggle de estado inline, y tres tipos de acciones por fila (editar, reabastecer,
eliminar). Todas las piezas construidas en este módulo se ensamblan aquí.

```tsx
// src/presentation/pages/admin/AdminProductsPage.tsx
import { useEffect, useState, useCallback, useRef } from "react";
import {
  Pencil,
  Trash2,
  Plus,
  Package,
  Search,
  ChevronLeft,
  ChevronRight,
  BoxSelect,
} from "lucide-react";
import { AdminShell } from "@/presentation/components/AdminShell";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Badge } from "@/components/ui/badge";
import { Skeleton } from "@/components/ui/skeleton";
import { Switch } from "@/components/ui/switch";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { useToast } from "@/components/ui/use-toast";
import { useAdminStore } from "@/domain/store/admin.store";
import type { Product } from "@/domain/model/catalog.types";
import { ProductDialog } from "@/presentation/components/admin/ProductDialog";
import { RestockDialog } from "@/presentation/components/admin/RestockDialog";
import { DeleteConfirmDialog } from "@/presentation/components/admin/DeleteConfirmDialog";
import { formatPrice } from "@/core/utils/formatters";

// ── Constantes ────────────────────────────────────────────────────────────────

const PAGE_SIZE = 10;
const DEBOUNCE_MS = 400;

// ── Componente ────────────────────────────────────────────────────────────────

export function AdminProductsPage() {
  const { toast } = useToast();

  // Store
  const adminProducts = useAdminStore((s) => s.adminProducts);
  const productsLoading = useAdminStore((s) => s.productsLoading);
  const productsError = useAdminStore((s) => s.productsError);
  const productsTotal = useAdminStore((s) => s.productsTotal);
  const fetchAdminProducts = useAdminStore((s) => s.fetchAdminProducts);
  const updateProduct = useAdminStore((s) => s.updateProduct);
  const deleteProduct = useAdminStore((s) => s.deleteProduct);
  const categories = useAdminStore((s) => s.categories);
  const fetchAdminCategories = useAdminStore((s) => s.fetchAdminCategories);

  // Estado local
  const [page, setPage] = useState(1);
  const [search, setSearch] = useState("");
  const [debouncedSearch, setDebouncedSearch] = useState("");
  const debounceRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  // Estado de diálogos
  const [createOpen, setCreateOpen] = useState(false);
  const [editTarget, setEditTarget] = useState<Product | null>(null);
  const [restockTarget, setRestockTarget] = useState<Product | null>(null);
  const [deleteTarget, setDeleteTarget] = useState<Product | null>(null);
  const [isDeleting, setIsDeleting] = useState(false);
  const [togglingId, setTogglingId] = useState<number | null>(null);

  // Paginación
  const totalPages = Math.ceil(productsTotal / PAGE_SIZE);

  // ── Efectos ───────────────────────────────────────────────────────────────

  // Carga inicial de categorías (necesarias para los formularios)
  useEffect(() => {
    if (categories.length === 0) {
      fetchAdminCategories();
    }
  }, [fetchAdminCategories, categories.length]);

  // Debounce del buscador
  useEffect(() => {
    if (debounceRef.current) clearTimeout(debounceRef.current);
    debounceRef.current = setTimeout(() => {
      setDebouncedSearch(search);
      setPage(1); // Resetea a página 1 al buscar
    }, DEBOUNCE_MS);
    return () => {
      if (debounceRef.current) clearTimeout(debounceRef.current);
    };
  }, [search]);

  // Carga de productos cuando cambia página o búsqueda
  useEffect(() => {
    fetchAdminProducts(page, debouncedSearch);
  }, [fetchAdminProducts, page, debouncedSearch]);

  // ── Handlers ──────────────────────────────────────────────────────────────

  const handleToggleActive = useCallback(
    async (product: Product) => {
      setTogglingId(product.id);
      try {
        await updateProduct(product.id, { is_active: !product.is_active });
        toast({
          title: product.is_active ? "Producto desactivado" : "Producto activado",
          description: `"${product.name}" fue ${product.is_active ? "ocultado" : "publicado"} en el catálogo.`,
        });
      } catch {
        toast({
          variant: "destructive",
          title: "Error",
          description: "No se pudo cambiar el estado del producto.",
        });
      } finally {
        setTogglingId(null);
      }
    },
    [updateProduct, toast]
  );

  const handleDelete = async () => {
    if (!deleteTarget) return;
    setIsDeleting(true);
    try {
      await deleteProduct(deleteTarget.id);
      toast({
        title: "Producto eliminado",
        description: `"${deleteTarget.name}" fue eliminado correctamente.`,
      });
      setDeleteTarget(null);
    } catch {
      toast({
        variant: "destructive",
        title: "Error",
        description: "No se pudo eliminar el producto.",
      });
    } finally {
      setIsDeleting(false);
    }
  };

  // ── Render ────────────────────────────────────────────────────────────────

  return (
    <AdminShell>
      {/* Encabezado */}
      <div className="flex items-center justify-between mb-6">
        <div>
          <h1 className="text-2xl font-bold tracking-tight">Productos</h1>
          <p className="text-muted-foreground text-sm mt-1">
            Gestiona el catálogo completo de productos.
          </p>
        </div>
        <Button onClick={() => setCreateOpen(true)}>
          <Plus className="mr-2 h-4 w-4" />
          Nuevo producto
        </Button>
      </div>

      {/* Buscador */}
      <div className="relative mb-4 max-w-sm">
        <Search className="absolute left-2.5 top-2.5 h-4 w-4 text-muted-foreground" />
        <Input
          className="pl-9"
          placeholder="Buscar por nombre..."
          value={search}
          onChange={(e) => setSearch(e.target.value)}
        />
      </div>

      {/* Error */}
      {productsError && (
        <div className="rounded-md bg-destructive/10 border border-destructive/20 text-destructive px-4 py-3 text-sm mb-4">
          {productsError}
        </div>
      )}

      {/* Tabla */}
      <div className="rounded-md border">
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead className="w-16">Imagen</TableHead>
              <TableHead>Nombre</TableHead>
              <TableHead>Categoría</TableHead>
              <TableHead className="text-right">Precio</TableHead>
              <TableHead className="text-center">Stock</TableHead>
              <TableHead className="text-center">Estado</TableHead>
              <TableHead className="text-right">Acciones</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {/* Skeletons */}
            {productsLoading &&
              Array.from({ length: PAGE_SIZE }).map((_, i) => (
                <TableRow key={`skeleton-${i}`}>
                  <TableCell>
                    <Skeleton className="h-10 w-10 rounded" />
                  </TableCell>
                  <TableCell>
                    <Skeleton className="h-4 w-40" />
                  </TableCell>
                  <TableCell>
                    <Skeleton className="h-4 w-24" />
                  </TableCell>
                  <TableCell className="text-right">
                    <Skeleton className="h-4 w-16 ml-auto" />
                  </TableCell>
                  <TableCell className="text-center">
                    <Skeleton className="h-4 w-10 mx-auto" />
                  </TableCell>
                  <TableCell className="text-center">
                    <Skeleton className="h-5 w-9 mx-auto rounded-full" />
                  </TableCell>
                  <TableCell className="text-right">
                    <Skeleton className="h-8 w-28 ml-auto" />
                  </TableCell>
                </TableRow>
              ))}

            {/* Estado vacío */}
            {!productsLoading && adminProducts.length === 0 && (
              <TableRow>
                <TableCell colSpan={7} className="text-center py-12">
                  <div className="flex flex-col items-center gap-2 text-muted-foreground">
                    <BoxSelect className="h-10 w-10" />
                    <p className="text-sm font-medium">
                      {debouncedSearch
                        ? `Sin resultados para "${debouncedSearch}"`
                        : "No hay productos registrados"}
                    </p>
                    {!debouncedSearch && (
                      <Button
                        variant="outline"
                        size="sm"
                        onClick={() => setCreateOpen(true)}
                      >
                        <Plus className="mr-2 h-3 w-3" />
                        Crear primer producto
                      </Button>
                    )}
                  </div>
                </TableCell>
              </TableRow>
            )}

            {/* Filas de datos */}
            {!productsLoading &&
              adminProducts.map((product) => (
                <TableRow key={product.id}>
                  {/* Thumbnail */}
                  <TableCell>
                    {product.image ? (
                      <img
                        src={product.image}
                        alt={product.name}
                        className="h-10 w-10 rounded object-cover border"
                      />
                    ) : (
                      <div className="h-10 w-10 rounded border bg-muted flex items-center justify-center">
                        <Package className="h-4 w-4 text-muted-foreground" />
                      </div>
                    )}
                  </TableCell>

                  {/* Nombre */}
                  <TableCell className="font-medium max-w-[200px] truncate">
                    {product.name}
                  </TableCell>

                  {/* Categoría */}
                  <TableCell className="text-muted-foreground text-sm">
                    {product.category_name ?? "—"}
                  </TableCell>

                  {/* Precio */}
                  <TableCell className="text-right font-mono text-sm">
                    {formatPrice(product.price)}
                  </TableCell>

                  {/* Stock */}
                  <TableCell className="text-center">
                    <span
                      className={
                        product.stock === 0
                          ? "text-destructive font-semibold"
                          : "text-foreground"
                      }
                    >
                      {product.stock}
                    </span>
                  </TableCell>

                  {/* Estado — switch inline */}
                  <TableCell className="text-center">
                    <Switch
                      checked={product.is_active}
                      onCheckedChange={() => handleToggleActive(product)}
                      disabled={togglingId === product.id}
                      aria-label={
                        product.is_active
                          ? "Desactivar producto"
                          : "Activar producto"
                      }
                    />
                  </TableCell>

                  {/* Acciones */}
                  <TableCell className="text-right">
                    <div className="flex items-center justify-end gap-1">
                      {/* Editar */}
                      <Button
                        variant="ghost"
                        size="icon"
                        className="h-8 w-8"
                        onClick={() => setEditTarget(product)}
                        title="Editar producto"
                      >
                        <Pencil className="h-4 w-4" />
                        <span className="sr-only">Editar</span>
                      </Button>

                      {/* Restock */}
                      <Button
                        variant="ghost"
                        size="icon"
                        className="h-8 w-8"
                        onClick={() => setRestockTarget(product)}
                        title="Reabastecer stock"
                      >
                        <Package className="h-4 w-4" />
                        <span className="sr-only">Reabastecer</span>
                      </Button>

                      {/* Eliminar */}
                      <Button
                        variant="ghost"
                        size="icon"
                        className="h-8 w-8 text-destructive hover:text-destructive hover:bg-destructive/10"
                        onClick={() => setDeleteTarget(product)}
                        title="Eliminar producto"
                      >
                        <Trash2 className="h-4 w-4" />
                        <span className="sr-only">Eliminar</span>
                      </Button>
                    </div>
                  </TableCell>
                </TableRow>
              ))}
          </TableBody>
        </Table>
      </div>

      {/* Paginación */}
      {!productsLoading && totalPages > 1 && (
        <div className="flex items-center justify-between mt-4">
          <p className="text-sm text-muted-foreground">
            Mostrando{" "}
            <span className="font-medium">
              {(page - 1) * PAGE_SIZE + 1}–
              {Math.min(page * PAGE_SIZE, productsTotal)}
            </span>{" "}
            de <span className="font-medium">{productsTotal}</span> productos
          </p>
          <div className="flex items-center gap-2">
            <Button
              variant="outline"
              size="icon"
              className="h-8 w-8"
              onClick={() => setPage((p) => Math.max(1, p - 1))}
              disabled={page === 1}
            >
              <ChevronLeft className="h-4 w-4" />
              <span className="sr-only">Página anterior</span>
            </Button>
            <span className="text-sm">
              {page} / {totalPages}
            </span>
            <Button
              variant="outline"
              size="icon"
              className="h-8 w-8"
              onClick={() => setPage((p) => Math.min(totalPages, p + 1))}
              disabled={page === totalPages}
            >
              <ChevronRight className="h-4 w-4" />
              <span className="sr-only">Página siguiente</span>
            </Button>
          </div>
        </div>
      )}

      {/* Diálogo de creación */}
      <ProductDialog
        open={createOpen}
        onOpenChange={setCreateOpen}
        categories={categories}
      />

      {/* Diálogo de edición */}
      <ProductDialog
        open={Boolean(editTarget)}
        onOpenChange={(open) => {
          if (!open) setEditTarget(null);
        }}
        product={editTarget ?? undefined}
        categories={categories}
      />

      {/* Diálogo de restock */}
      {restockTarget && (
        <RestockDialog
          open={Boolean(restockTarget)}
          onOpenChange={(open) => {
            if (!open) setRestockTarget(null);
          }}
          product={restockTarget}
        />
      )}

      {/* Diálogo de confirmación de eliminación */}
      <DeleteConfirmDialog
        open={Boolean(deleteTarget)}
        onOpenChange={(open) => {
          if (!open) setDeleteTarget(null);
        }}
        title="¿Eliminar producto?"
        description={
          deleteTarget
            ? `Esta acción no se puede deshacer. El producto "${deleteTarget.name}" será eliminado permanentemente del catálogo.`
            : ""
        }
        onConfirm={handleDelete}
        isLoading={isDeleting}
      />
    </AdminShell>
  );
}
```

### Utilidad `formatPrice` en core/utils

La función `formatPrice` usada en la tabla debe existir en `src/core/utils/formatters.ts`.
Si aún no la tienes, agrégala:

```typescript
// src/core/utils/formatters.ts

/**
 * Formatea un número como precio en dólares.
 * Ej.: formatPrice(1299.9) → "$1,299.90"
 */
export function formatPrice(value: number): string {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
    minimumFractionDigits: 2,
  }).format(value);
}

/**
 * Formatea una fecha ISO o Date a formato legible.
 * Ej.: formatDate("2024-03-15T10:30:00Z") → "15 mar 2024"
 */
export function formatDate(value: string | Date): string {
  const date = typeof value === "string" ? new Date(value) : value;
  return new Intl.DateTimeFormat("es-ES", {
    day: "numeric",
    month: "short",
    year: "numeric",
  }).format(date);
}
```

### Registrar la ruta en el router

```tsx
// src/main.tsx (o donde estén definidas tus rutas)
import { AdminProductsPage } from "@/presentation/pages/admin/AdminProductsPage";

// Dentro de las rutas protegidas de admin:
{
  path: "/admin/products",
  element: <AdminProductsPage />,
}
```

Agrega el enlace en la barra lateral de `AdminShell`:

```tsx
// Dentro de AdminShell.tsx — listado de navegación
{
  label: "Productos",
  href: "/admin/products",
  icon: Package,
}
```

---

## 11.7 Checkpoint y tabla resumen

Ejecuta el servidor de desarrollo y verifica cada flujo:

```bash
npm run dev
```

**Flujos a probar:**

1. Navega a `http://localhost:5173/admin/products` — la tabla carga con skeletons y luego muestra los productos.
2. Escribe en el buscador — los resultados se filtran después de 400 ms sin recargar la página.
3. Navega con los botones de paginación — la URL no cambia pero los datos sí.
4. Clic en "Nuevo producto" — el diálogo se abre. Prueba guardar con campos vacíos — los errores de validación aparecen campo por campo.
5. Crea un producto válido — aparece al inicio de la lista con el toast de confirmación.
6. Clic en editar (ícono lápiz) — el diálogo se abre con los datos del producto. Modifica el precio y guarda.
7. Clic en reabastecer (ícono caja) — el diálogo muestra el stock actual. Agrega unidades y confirma.
8. Usa el switch en la fila — el estado activo/inactivo cambia inline con el toast correspondiente.
9. Clic en eliminar — el AlertDialog de confirmación aparece. Confirma — el producto desaparece y el total se actualiza.
10. Verifica que `DeleteConfirmDialog` importado desde el módulo 10 funciona sin modificaciones.

---

### Tabla resumen del módulo

| Elemento | Archivo | Descripción |
|---|---|---|
| API CRUD productos | `src/data/api/admin.api.ts` | `getAdminProducts`, `createProduct`, `updateProduct`, `deleteProduct`, `restockProduct` |
| Tipo `PaginatedProducts` | `src/data/api/admin.api.ts` | Modela la respuesta paginada de DRF |
| Admin Store — slice productos | `src/domain/store/admin.store.ts` | Estado con `adminProducts`, `productsTotal` y 5 acciones |
| `ProductForm` | `src/presentation/components/admin/ProductForm.tsx` | Formulario Zod con 6 campos incluyendo Select y Switch |
| `ProductDialog` | `src/presentation/components/admin/ProductDialog.tsx` | Modal grande con scroll para crear/editar |
| `RestockDialog` | `src/presentation/components/admin/RestockDialog.tsx` | Diálogo enfocado para añadir unidades al stock |
| `AdminProductsPage` | `src/presentation/pages/admin/AdminProductsPage.tsx` | Página con búsqueda debounced, paginación y 3 acciones por fila |
| `DeleteConfirmDialog` | `src/presentation/components/admin/DeleteConfirmDialog.tsx` | Reutilizado del Módulo 10 sin cambios |
| `formatPrice` | `src/core/utils/formatters.ts` | Formatea precios con `Intl.NumberFormat` |
| Ruta | `src/main.tsx` | `/admin/products` protegida |

### Tipos de `catalog.types.ts` usados en este módulo

```typescript
// src/domain/model/catalog.types.ts (referencia — no modificar)
export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  stock: number;
  category: number;
  category_name: string;
  image: string | null;
  is_active: boolean;
}

export interface Category {
  id: number;
  name: string;
  description: string;
  product_count?: number;
}
```

### Resumen de acciones del AdminStore al final del Módulo 11

```
useAdminStore
├── Categorías
│   ├── fetchAdminCategories()
│   ├── createCategory(data)
│   ├── updateCategory(id, data)
│   └── deleteCategory(id)
└── Productos
    ├── fetchAdminProducts(page?, search?)
    ├── createProduct(data)
    ├── updateProduct(id, data)
    ├── deleteProduct(id)
    └── restockProduct(id, qty)
```
