# ShopApp React Web — Módulo 10

## Admin Categorías — CRUD completo con tabla y diálogos

**Duración estimada:** 3 horas
**Prerequisitos:** Módulos 1–9

---

> **Objetivo**
> Implementar el panel de administración de categorías con operaciones CRUD completas. Se extenderán
> las funciones de API de admin, el store de Zustand para categorías, y se construirán los componentes
> de tabla, diálogo de creación/edición y confirmación de eliminación, todo integrado en una página
> protegida de admin.
>
> **Checkpoint final**
> - La ruta `/admin/categories` muestra una tabla con todas las categorías del backend.
> - El botón "Nueva categoría" abre un diálogo con formulario validado que crea una nueva categoría y actualiza la tabla.
> - El ícono de edición abre el diálogo pre-poblado y guarda cambios con PATCH.
> - El ícono de eliminar abre un AlertDialog de confirmación y borra la categoría con DELETE.
> - Los estados de carga muestran skeletons y los errores se manejan correctamente.

---

## 10.1 Funciones API de admin para categorías

El archivo `src/data/api/admin.api.ts` centraliza todas las llamadas HTTP que requieren autenticación
de staff. Se extiende con cuatro funciones CRUD para categorías, usando la instancia de Axios que
ya incluye el token de autorización configurado en módulos anteriores.

```typescript
// src/data/api/admin.api.ts
import apiClient from "@/data/api/client";
import type { Category, Product } from "@/domain/model/catalog.types";

// ── Tipos de payload ──────────────────────────────────────────────────────────

export interface CategoryPayload {
  name: string;
  description?: string;
}

export interface ProductPayload {
  name: string;
  description?: string;
  price: number;
  stock: number;
  category: number;
  is_active: boolean;
}

// ── Categorías ────────────────────────────────────────────────────────────────

/** GET /categories/ — lista todas las categorías (incluye product_count) */
export async function getCategories(): Promise<Category[]> {
  const response = await apiClient.get<Category[]>("/categories/");
  return response.data;
}

/** POST /categories/ — crea una nueva categoría */
export async function createCategory(data: CategoryPayload): Promise<Category> {
  const response = await apiClient.post<Category>("/categories/", data);
  return response.data;
}

/** PATCH /categories/{id}/ — actualiza parcialmente una categoría */
export async function updateCategory(
  id: number,
  data: Partial<CategoryPayload>
): Promise<Category> {
  const response = await apiClient.patch<Category>(`/categories/${id}/`, data);
  return response.data;
}

/** DELETE /categories/{id}/ — elimina una categoría */
export async function deleteCategory(id: number): Promise<void> {
  await apiClient.delete(`/categories/${id}/`);
}
```

> **Nota:** `apiClient` es la instancia de Axios configurada en `src/data/api/client.ts` con
> `baseURL` apuntando al backend Django y el interceptor que adjunta el token JWT. Si aún no tienes
> este archivo, revisa el Módulo 5.

---

## 10.2 AdminStore — estado y acciones de categorías

El store de Zustand para admin se extiende con un slice de categorías. Cada acción actualiza el
estado local después de confirmar el éxito en el backend, evitando actualizaciones optimistas
que podrían dejar el estado inconsistente ante errores de red.

```typescript
// src/domain/store/admin.store.ts
import { create } from "zustand";
import type { Category } from "@/domain/model/catalog.types";
import {
  getCategories,
  createCategory as apiCreateCategory,
  updateCategory as apiUpdateCategory,
  deleteCategory as apiDeleteCategory,
  type CategoryPayload,
} from "@/data/api/admin.api";

// ── Estado ────────────────────────────────────────────────────────────────────

interface AdminState {
  // Categorías
  categories: Category[];
  categoriesLoading: boolean;
  categoriesError: string | null;

  // Acciones de categorías
  fetchAdminCategories: () => Promise<void>;
  createCategory: (data: CategoryPayload) => Promise<void>;
  updateCategory: (id: number, data: Partial<CategoryPayload>) => Promise<void>;
  deleteCategory: (id: number) => Promise<void>;
}

// ── Store ─────────────────────────────────────────────────────────────────────

export const useAdminStore = create<AdminState>((set, get) => ({
  // Estado inicial — categorías
  categories: [],
  categoriesLoading: false,
  categoriesError: null,

  // ── fetchAdminCategories ──────────────────────────────────────────────────
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

  // ── createCategory ────────────────────────────────────────────────────────
  createCategory: async (data: CategoryPayload) => {
    const newCategory = await apiCreateCategory(data);
    set((state) => ({
      categories: [...state.categories, newCategory],
    }));
  },

  // ── updateCategory ────────────────────────────────────────────────────────
  updateCategory: async (id: number, data: Partial<CategoryPayload>) => {
    const updated = await apiUpdateCategory(id, data);
    set((state) => ({
      categories: state.categories.map((c) => (c.id === id ? updated : c)),
    }));
  },

  // ── deleteCategory ────────────────────────────────────────────────────────
  deleteCategory: async (id: number) => {
    await apiDeleteCategory(id);
    set((state) => ({
      categories: state.categories.filter((c) => c.id !== id),
    }));
  },
}));
```

> **Patrón:** `createCategory` y `updateCategory` lanzan el error hacia arriba si la API falla,
> permitiendo que el componente capture la excepción y muestre feedback al usuario sin modificar
> el estado local. Solo `fetchAdminCategories` captura el error internamente porque alimenta
> el estado de error visible en la UI.

---

## 10.3 Componente CategoryForm

`CategoryForm` es un formulario controlado por React Hook Form con validación Zod. Recibe
`defaultValues` opcionales para el modo edición y expone `onSubmit` como prop, dejando al
componente padre la responsabilidad de decidir si crear o actualizar.

```tsx
// src/presentation/components/admin/CategoryForm.tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Button } from "@/components/ui/button";
import { Loader2 } from "lucide-react";

// ── Schema Zod ────────────────────────────────────────────────────────────────

const categorySchema = z.object({
  name: z
    .string()
    .min(2, "El nombre debe tener al menos 2 caracteres")
    .max(100, "El nombre no puede superar los 100 caracteres"),
  description: z.string().max(500, "La descripción no puede superar los 500 caracteres").optional(),
});

export type CategoryFormValues = z.infer<typeof categorySchema>;

// ── Props ─────────────────────────────────────────────────────────────────────

interface CategoryFormProps {
  defaultValues?: Partial<CategoryFormValues>;
  onSubmit: (data: CategoryFormValues) => Promise<void>;
  isLoading?: boolean;
}

// ── Componente ────────────────────────────────────────────────────────────────

export function CategoryForm({
  defaultValues,
  onSubmit,
  isLoading = false,
}: CategoryFormProps) {
  const form = useForm<CategoryFormValues>({
    resolver: zodResolver(categorySchema),
    defaultValues: {
      name: defaultValues?.name ?? "",
      description: defaultValues?.description ?? "",
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
                  placeholder="Ej. Electrónica"
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
                  placeholder="Describe brevemente esta categoría..."
                  rows={3}
                  disabled={isLoading}
                  {...field}
                />
              </FormControl>
              <FormMessage />
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

---

## 10.4 Componente CategoryDialog

`CategoryDialog` envuelve `CategoryForm` en un diálogo modal de shadcn/ui. Detecta si recibe
una `category` como prop para determinar si debe crear o editar, y llama a la acción correcta
del store. Cierra el diálogo y llama a `onSuccess` tras una operación exitosa.

```tsx
// src/presentation/components/admin/CategoryDialog.tsx
import { useState } from "react";
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog";
import { useToast } from "@/components/ui/use-toast";
import type { Category } from "@/domain/model/catalog.types";
import { useAdminStore } from "@/domain/store/admin.store";
import { CategoryForm, type CategoryFormValues } from "./CategoryForm";

// ── Props ─────────────────────────────────────────────────────────────────────

interface CategoryDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  category?: Category;
  onSuccess?: () => void;
}

// ── Componente ────────────────────────────────────────────────────────────────

export function CategoryDialog({
  open,
  onOpenChange,
  category,
  onSuccess,
}: CategoryDialogProps) {
  const [isLoading, setIsLoading] = useState(false);
  const { toast } = useToast();
  const createCategory = useAdminStore((s) => s.createCategory);
  const updateCategory = useAdminStore((s) => s.updateCategory);

  const isEditing = Boolean(category);
  const title = isEditing ? "Editar categoría" : "Nueva categoría";

  const handleSubmit = async (data: CategoryFormValues) => {
    setIsLoading(true);
    try {
      if (isEditing && category) {
        await updateCategory(category.id, data);
        toast({
          title: "Categoría actualizada",
          description: `"${data.name}" fue actualizada correctamente.`,
        });
      } else {
        await createCategory(data);
        toast({
          title: "Categoría creada",
          description: `"${data.name}" fue creada correctamente.`,
        });
      }
      onOpenChange(false);
      onSuccess?.();
    } catch {
      toast({
        variant: "destructive",
        title: "Error",
        description: isEditing
          ? "No se pudo actualizar la categoría."
          : "No se pudo crear la categoría.",
      });
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent className="sm:max-w-md">
        <DialogHeader>
          <DialogTitle>{title}</DialogTitle>
        </DialogHeader>
        <CategoryForm
          defaultValues={
            category
              ? { name: category.name, description: category.description }
              : undefined
          }
          onSubmit={handleSubmit}
          isLoading={isLoading}
        />
      </DialogContent>
    </Dialog>
  );
}
```

---

## 10.5 Componente DeleteConfirmDialog

`DeleteConfirmDialog` es un AlertDialog reutilizable para confirmar cualquier operación de
eliminación en el panel de admin. Recibe texto personalizable para el título y la descripción,
y un callback `onConfirm` que puede ser asíncrono, mostrando el estado de carga en el botón
de confirmación.

```tsx
// src/presentation/components/admin/DeleteConfirmDialog.tsx
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { Loader2 } from "lucide-react";

// ── Props ─────────────────────────────────────────────────────────────────────

interface DeleteConfirmDialogProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  title: string;
  description: string;
  onConfirm: () => Promise<void> | void;
  isLoading?: boolean;
}

// ── Componente ────────────────────────────────────────────────────────────────

export function DeleteConfirmDialog({
  open,
  onOpenChange,
  title,
  description,
  onConfirm,
  isLoading = false,
}: DeleteConfirmDialogProps) {
  const handleConfirm = async (e: React.MouseEvent) => {
    // Prevent AlertDialog from auto-closing before async completes
    e.preventDefault();
    await onConfirm();
  };

  return (
    <AlertDialog open={open} onOpenChange={onOpenChange}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{title}</AlertDialogTitle>
          <AlertDialogDescription>{description}</AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel disabled={isLoading}>Cancelar</AlertDialogCancel>
          <AlertDialogAction
            onClick={handleConfirm}
            disabled={isLoading}
            className="bg-destructive text-destructive-foreground hover:bg-destructive/90"
          >
            {isLoading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
            {isLoading ? "Eliminando..." : "Eliminar"}
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

> **Nota sobre `e.preventDefault()`:** El componente `AlertDialogAction` de shadcn cierra el
> diálogo automáticamente al hacer clic. Prevenimos este comportamiento predeterminado para
> que el diálogo permanezca abierto mientras la operación asíncrona se completa, y el componente
> padre controle el cierre llamando a `onOpenChange(false)` al terminar.

---

## 10.6 Página AdminCategoriesPage

`AdminCategoriesPage` orquesta todos los componentes anteriores en una página completa de
administración. Carga las categorías al montar, gestiona el estado de los diálogos localmente,
y renderiza la tabla con shadcn/ui incluyendo esqueletos de carga y un estado vacío.

```tsx
// src/presentation/pages/admin/AdminCategoriesPage.tsx
import { useEffect, useState } from "react";
import { Pencil, Trash2, Plus, FolderOpen } from "lucide-react";
import { AdminShell } from "@/presentation/components/AdminShell";
import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";
import { Skeleton } from "@/components/ui/skeleton";
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
import type { Category } from "@/domain/model/catalog.types";
import { CategoryDialog } from "@/presentation/components/admin/CategoryDialog";
import { DeleteConfirmDialog } from "@/presentation/components/admin/DeleteConfirmDialog";

// ── Componente ────────────────────────────────────────────────────────────────

export function AdminCategoriesPage() {
  const { toast } = useToast();

  // Store
  const categories = useAdminStore((s) => s.categories);
  const categoriesLoading = useAdminStore((s) => s.categoriesLoading);
  const categoriesError = useAdminStore((s) => s.categoriesError);
  const fetchAdminCategories = useAdminStore((s) => s.fetchAdminCategories);
  const deleteCategory = useAdminStore((s) => s.deleteCategory);

  // Estado local de diálogos
  const [createOpen, setCreateOpen] = useState(false);
  const [editTarget, setEditTarget] = useState<Category | null>(null);
  const [deleteTarget, setDeleteTarget] = useState<Category | null>(null);
  const [isDeleting, setIsDeleting] = useState(false);

  // Carga inicial
  useEffect(() => {
    fetchAdminCategories();
  }, [fetchAdminCategories]);

  // ── Handlers ─────────────────────────────────────────────────────────────

  const handleDelete = async () => {
    if (!deleteTarget) return;
    setIsDeleting(true);
    try {
      await deleteCategory(deleteTarget.id);
      toast({
        title: "Categoría eliminada",
        description: `"${deleteTarget.name}" fue eliminada correctamente.`,
      });
      setDeleteTarget(null);
    } catch {
      toast({
        variant: "destructive",
        title: "Error",
        description: "No se pudo eliminar la categoría.",
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
          <h1 className="text-2xl font-bold tracking-tight">Categorías</h1>
          <p className="text-muted-foreground text-sm mt-1">
            Gestiona las categorías del catálogo.
          </p>
        </div>
        <Button onClick={() => setCreateOpen(true)}>
          <Plus className="mr-2 h-4 w-4" />
          Nueva categoría
        </Button>
      </div>

      {/* Error */}
      {categoriesError && (
        <div className="rounded-md bg-destructive/10 border border-destructive/20 text-destructive px-4 py-3 text-sm mb-4">
          {categoriesError}
        </div>
      )}

      {/* Tabla */}
      <div className="rounded-md border">
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Nombre</TableHead>
              <TableHead>Descripción</TableHead>
              <TableHead className="text-center">Productos</TableHead>
              <TableHead className="text-right">Acciones</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {/* Skeletons de carga */}
            {categoriesLoading &&
              Array.from({ length: 5 }).map((_, i) => (
                <TableRow key={`skeleton-${i}`}>
                  <TableCell>
                    <Skeleton className="h-4 w-32" />
                  </TableCell>
                  <TableCell>
                    <Skeleton className="h-4 w-48" />
                  </TableCell>
                  <TableCell className="text-center">
                    <Skeleton className="h-4 w-8 mx-auto" />
                  </TableCell>
                  <TableCell className="text-right">
                    <Skeleton className="h-8 w-20 ml-auto" />
                  </TableCell>
                </TableRow>
              ))}

            {/* Estado vacío */}
            {!categoriesLoading && categories.length === 0 && (
              <TableRow>
                <TableCell colSpan={4} className="text-center py-12">
                  <div className="flex flex-col items-center gap-2 text-muted-foreground">
                    <FolderOpen className="h-10 w-10" />
                    <p className="text-sm font-medium">
                      No hay categorías registradas
                    </p>
                    <Button
                      variant="outline"
                      size="sm"
                      onClick={() => setCreateOpen(true)}
                    >
                      <Plus className="mr-2 h-3 w-3" />
                      Crear primera categoría
                    </Button>
                  </div>
                </TableCell>
              </TableRow>
            )}

            {/* Filas de datos */}
            {!categoriesLoading &&
              categories.map((category) => (
                <TableRow key={category.id}>
                  <TableCell className="font-medium">{category.name}</TableCell>
                  <TableCell className="text-muted-foreground max-w-xs truncate">
                    {category.description || (
                      <span className="italic text-xs">Sin descripción</span>
                    )}
                  </TableCell>
                  <TableCell className="text-center">
                    <Badge variant="secondary">
                      {category.product_count ?? 0}
                    </Badge>
                  </TableCell>
                  <TableCell className="text-right">
                    <div className="flex items-center justify-end gap-1">
                      {/* Editar */}
                      <Button
                        variant="ghost"
                        size="icon"
                        className="h-8 w-8"
                        onClick={() => setEditTarget(category)}
                        title="Editar categoría"
                      >
                        <Pencil className="h-4 w-4" />
                        <span className="sr-only">Editar</span>
                      </Button>

                      {/* Eliminar */}
                      <Button
                        variant="ghost"
                        size="icon"
                        className="h-8 w-8 text-destructive hover:text-destructive hover:bg-destructive/10"
                        onClick={() => setDeleteTarget(category)}
                        title="Eliminar categoría"
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

      {/* Diálogo de creación */}
      <CategoryDialog
        open={createOpen}
        onOpenChange={setCreateOpen}
      />

      {/* Diálogo de edición */}
      <CategoryDialog
        open={Boolean(editTarget)}
        onOpenChange={(open) => {
          if (!open) setEditTarget(null);
        }}
        category={editTarget ?? undefined}
      />

      {/* Diálogo de confirmación de eliminación */}
      <DeleteConfirmDialog
        open={Boolean(deleteTarget)}
        onOpenChange={(open) => {
          if (!open) setDeleteTarget(null);
        }}
        title="¿Eliminar categoría?"
        description={
          deleteTarget
            ? `Esta acción no se puede deshacer. La categoría "${deleteTarget.name}" será eliminada permanentemente. Los productos asociados quedarán sin categoría.`
            : ""
        }
        onConfirm={handleDelete}
        isLoading={isDeleting}
      />
    </AdminShell>
  );
}
```

### Registrar la ruta en el router

Agrega la nueva página al router de la aplicación, protegida por el guard de autenticación
de staff que ya existe desde módulos anteriores:

```tsx
// src/main.tsx (o donde estén definidas tus rutas)
import { AdminCategoriesPage } from "@/presentation/pages/admin/AdminCategoriesPage";

// Dentro de las rutas protegidas de admin:
{
  path: "/admin/categories",
  element: <AdminCategoriesPage />,
}
```

Agrega el enlace en `AdminShell` o en la barra lateral de navegación de admin:

```tsx
// Dentro de AdminShell.tsx — listado de navegación
{
  label: "Categorías",
  href: "/admin/categories",
  icon: FolderOpen,
}
```

---

## 10.7 Checkpoint y tabla resumen

Verifica que todo funciona ejecutando el servidor de desarrollo y probando cada flujo:

```bash
npm run dev
```

**Flujos a probar:**

1. Navega a `http://localhost:5173/admin/categories` — debe mostrar la tabla con skeletons y luego los datos.
2. Clic en "Nueva categoría" — se abre el diálogo. Intenta guardar sin nombre — debe mostrar el error de validación.
3. Ingresa nombre y crea — aparece en la tabla y se muestra el toast de confirmación.
4. Clic en el ícono de edición — el diálogo se abre con los datos pre-poblados. Modifica y guarda.
5. Clic en el ícono de eliminar — aparece el AlertDialog. Confirma — la fila desaparece.

---

### Tabla resumen del módulo

| Elemento | Archivo | Descripción |
|---|---|---|
| API CRUD categorías | `src/data/api/admin.api.ts` | `getCategories`, `createCategory`, `updateCategory`, `deleteCategory` |
| Admin Store — slice categorías | `src/domain/store/admin.store.ts` | Estado y 4 acciones con manejo de errores |
| `CategoryForm` | `src/presentation/components/admin/CategoryForm.tsx` | Formulario con React Hook Form + Zod |
| `CategoryDialog` | `src/presentation/components/admin/CategoryDialog.tsx` | Modal de creación y edición |
| `DeleteConfirmDialog` | `src/presentation/components/admin/DeleteConfirmDialog.tsx` | AlertDialog reutilizable para confirmaciones destructivas |
| `AdminCategoriesPage` | `src/presentation/pages/admin/AdminCategoriesPage.tsx` | Página con tabla, skeletons y estado vacío |
| Ruta | `src/main.tsx` | `/admin/categories` protegida |

### Tipos de `catalog.types.ts` usados en este módulo

```typescript
// src/domain/model/catalog.types.ts (referencia — no modificar)
export interface Category {
  id: number;
  name: string;
  description: string;
  product_count?: number;
}
```
