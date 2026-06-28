# ShopApp React Web — Módulo 14

## Imágenes — Subida de fotos de producto y avatar de usuario

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulos 1–13 completados. Backend `shopapi_07` desplegado con campo `image` en `Product` y campo `avatar` en `UserProfile`.

---

> **Objetivo**
> Implementar la subida de archivos de imagen tanto para fotos de producto (solo staff) como para el
> avatar del usuario autenticado. Se construye un componente reutilizable `ImageUploader` que valida
> tamaño, muestra vista previa local antes de enviar, y notifica el resultado. La integración usa
> `multipart/form-data` mediante Axios y `FormData` nativo.
>
> **Checkpoint final**
> - Un administrador puede subir una foto a cualquier producto desde el panel admin.
> - La imagen del producto aparece en `ProductCard` y en `ProductDetailPage` con fallback a placeholder.
> - El usuario puede cambiar su avatar desde `ProfilePage`.
> - El componente `UserAvatar` refleja el nuevo avatar inmediatamente tras la subida.
> - `npm run build` compila sin errores de TypeScript.

---

## 14.1 Image API

**Archivo:** `src/data/api/images.api.ts`

Este archivo centraliza las dos operaciones de subida de imagen que expone el backend. Ambas usan `PATCH` porque actualizan un recurso existente enviando únicamente el campo de imagen. Axios detecta automáticamente que el cuerpo es una instancia de `FormData` y establece el encabezado `Content-Type: multipart/form-data` con el `boundary` correcto — no es necesario configurarlo manualmente.

```typescript
// src/data/api/images.api.ts

import { apiClient } from './api.client';
import type { Product } from '../domain/model/catalog.types';
import type { UserProfile } from '../domain/model/user.types';

/**
 * Sube una imagen para un producto existente.
 * El backend espera un campo llamado "image" en el formulario multipart.
 *
 * @param productId - ID del producto a actualizar.
 * @param file - Archivo de imagen seleccionado por el usuario.
 * @returns El producto actualizado con la nueva URL de imagen.
 */
export async function uploadProductImage(
  productId: number,
  file: File
): Promise<Product> {
  const formData = new FormData();
  formData.append('image', file);

  const response = await apiClient.patch<Product>(
    `/products/${productId}/`,
    formData
    // No se incluye cabecera Content-Type: Axios la genera con el boundary correcto
  );
  return response.data;
}

/**
 * Sube un avatar para el usuario autenticado actualmente.
 * El backend espera un campo llamado "avatar" en el formulario multipart.
 *
 * @param file - Archivo de imagen seleccionado por el usuario.
 * @returns El perfil actualizado con la nueva URL de avatar.
 */
export async function uploadUserAvatar(file: File): Promise<UserProfile> {
  const formData = new FormData();
  formData.append('avatar', file);

  const response = await apiClient.patch<UserProfile>(
    '/users/me/',
    formData
  );
  return response.data;
}
```

**Por qué `PATCH` y no `PUT`:** `PATCH` envía solo los campos que cambian. Enviar `PUT` requeriría incluir todos los campos del recurso, lo que generaría errores de validación en los campos requeridos que no se están modificando (name, price, etc.).

---

## 14.2 Componente ImageUploader

**Archivo:** `src/presentation/components/ImageUploader.tsx`

`ImageUploader` es un componente reutilizable y sin estado de dominio. Recibe la URL de imagen actual (si existe), un callback `onUpload` que el padre debe proveer, y parámetros de configuración opcionales. La lógica del input nativo de archivo se oculta con `display: none` y se activa mediante un `<label>` estilizado, lo que permite diseño libre sin las limitaciones visuales del input nativo.

El flujo de usuario es:
1. El usuario hace clic en el área de imagen o en el botón "Cambiar imagen".
2. Se abre el selector de archivos del sistema operativo.
3. Al seleccionar un archivo: validación de tamaño, vista previa local inmediata con `URL.createObjectURL`.
4. El usuario hace clic en "Subir imagen" para confirmar la subida.
5. Se muestra spinner durante la petición y toast de éxito o mensaje de error al finalizar.

```typescript
// src/presentation/components/ImageUploader.tsx

import { useRef, useState } from 'react';
import { Upload, ImageIcon, X, Loader2 } from 'lucide-react';
import { Button } from '@/components/ui/button';
import { toast } from '@/components/ui/use-toast';
import { cn } from '@/lib/utils';

interface ImageUploaderProps {
  /** URL de la imagen actual almacenada en el backend. Puede ser null si no existe. */
  currentImageUrl?: string | null;
  /** Callback que el padre ejecuta con el archivo seleccionado. Debe retornar una Promise. */
  onUpload: (file: File) => Promise<void>;
  /** Tipos MIME aceptados. Por defecto solo imágenes. */
  accept?: string;
  /** Tamaño máximo permitido en megabytes. Por defecto 5 MB. */
  maxSizeMB?: number;
  /** Clases CSS adicionales para el contenedor externo. */
  className?: string;
  /** Cuando es true, aplica estilo circular (para avatares). */
  circular?: boolean;
}

export function ImageUploader({
  currentImageUrl,
  onUpload,
  accept = 'image/*',
  maxSizeMB = 5,
  className,
  circular = false,
}: ImageUploaderProps) {
  // URL de vista previa local. Puede ser la URL del backend o un blob local.
  const [previewUrl, setPreviewUrl] = useState<string | null>(
    currentImageUrl ?? null
  );
  // Archivo pendiente de subir (seleccionado pero no enviado aún).
  const [pendingFile, setPendingFile] = useState<File | null>(null);
  const [isUploading, setIsUploading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  // Referencia al input nativo, oculto visualmente.
  const inputRef = useRef<HTMLInputElement>(null);

  const maxSizeBytes = maxSizeMB * 1024 * 1024;

  /** Abre el selector de archivos del sistema operativo. */
  function handleClickArea() {
    inputRef.current?.click();
  }

  /** Procesa el archivo elegido: valida tamaño y genera vista previa local. */
  function handleFileChange(event: React.ChangeEvent<HTMLInputElement>) {
    const file = event.target.files?.[0];
    if (!file) return;

    setError(null);

    if (file.size > maxSizeBytes) {
      setError(`El archivo supera el límite de ${maxSizeMB} MB.`);
      // Limpia el input para permitir elegir de nuevo el mismo archivo.
      event.target.value = '';
      return;
    }

    // Revoca el blob anterior para liberar memoria antes de crear uno nuevo.
    if (previewUrl && previewUrl.startsWith('blob:')) {
      URL.revokeObjectURL(previewUrl);
    }

    const objectUrl = URL.createObjectURL(file);
    setPreviewUrl(objectUrl);
    setPendingFile(file);
  }

  /** Envía el archivo al backend mediante el callback onUpload del padre. */
  async function handleUpload() {
    if (!pendingFile) return;

    setIsUploading(true);
    setError(null);

    try {
      await onUpload(pendingFile);
      setPendingFile(null);
      toast({
        title: 'Imagen actualizada',
        description: 'La imagen se subió correctamente.',
      });
    } catch (err) {
      const message =
        err instanceof Error ? err.message : 'Error al subir la imagen.';
      setError(message);
      toast({
        title: 'Error',
        description: message,
        variant: 'destructive',
      });
    } finally {
      setIsUploading(false);
    }
  }

  /** Descarta el archivo pendiente y restaura la imagen original del backend. */
  function handleCancel() {
    if (previewUrl && previewUrl.startsWith('blob:')) {
      URL.revokeObjectURL(previewUrl);
    }
    setPreviewUrl(currentImageUrl ?? null);
    setPendingFile(null);
    setError(null);
    if (inputRef.current) {
      inputRef.current.value = '';
    }
  }

  return (
    <div className={cn('flex flex-col items-center gap-3', className)}>
      {/* Área clickeable que muestra la imagen o el placeholder */}
      <button
        type="button"
        onClick={handleClickArea}
        className={cn(
          'relative overflow-hidden border-2 border-dashed border-muted-foreground/30',
          'hover:border-primary/50 transition-colors cursor-pointer bg-muted',
          'focus:outline-none focus:ring-2 focus:ring-primary focus:ring-offset-2',
          circular
            ? 'w-28 h-28 rounded-full'
            : 'w-full aspect-square rounded-lg max-w-xs'
        )}
        aria-label="Seleccionar imagen"
      >
        {previewUrl ? (
          <>
            <img
              src={previewUrl}
              alt="Vista previa"
              className="w-full h-full object-cover"
              onError={() => setPreviewUrl(null)}
            />
            {/* Capa de hover con icono de cambio */}
            <div className="absolute inset-0 bg-black/40 opacity-0 hover:opacity-100 transition-opacity flex items-center justify-center">
              <Upload className="w-6 h-6 text-white" />
            </div>
          </>
        ) : (
          <div className="flex flex-col items-center justify-center h-full gap-2 p-4">
            <ImageIcon className="w-10 h-10 text-muted-foreground" />
            <span className="text-xs text-muted-foreground text-center">
              Haz clic para seleccionar
            </span>
          </div>
        )}
      </button>

      {/* Input nativo oculto — nunca se muestra directamente */}
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        onChange={handleFileChange}
        className="hidden"
        aria-hidden="true"
      />

      {/* Controles: solo visibles cuando hay un archivo pendiente */}
      {pendingFile && (
        <div className="flex gap-2 w-full max-w-xs">
          <Button
            type="button"
            onClick={handleUpload}
            disabled={isUploading}
            size="sm"
            className="flex-1"
          >
            {isUploading ? (
              <>
                <Loader2 className="w-4 h-4 mr-2 animate-spin" />
                Subiendo…
              </>
            ) : (
              <>
                <Upload className="w-4 h-4 mr-2" />
                Subir imagen
              </>
            )}
          </Button>
          <Button
            type="button"
            onClick={handleCancel}
            disabled={isUploading}
            variant="outline"
            size="sm"
          >
            <X className="w-4 h-4" />
          </Button>
        </div>
      )}

      {/* Texto informativo de tamaño máximo */}
      {!pendingFile && (
        <p className="text-xs text-muted-foreground">
          Máximo {maxSizeMB} MB · JPG, PNG, WebP
        </p>
      )}

      {/* Mensaje de error */}
      {error && (
        <p className="text-xs text-destructive text-center max-w-xs">{error}</p>
      )}
    </div>
  );
}
```

**Gestión de memoria:** cada vez que se genera un `blob:` URL con `URL.createObjectURL`, se revoca el anterior con `URL.revokeObjectURL` para evitar fugas de memoria. El navegador mantiene una referencia al blob en memoria hasta que se revoca explícitamente.

---

## 14.3 Integración en AdminProductsPage

**Archivo:** `src/presentation/pages/admin/AdminProductsPage.tsx` (fragmento actualizado)

Se muestra únicamente el diálogo de producto con las secciones nuevas. El resto de la página (tabla, paginación, botón de creación) no cambia respecto al módulo anterior.

La subida de imagen ocurre de forma independiente al formulario del producto. Al confirmar el archivo, la imagen se sube inmediatamente (sin esperar al submit del formulario). Este enfoque es más sencillo y evita mezclar lógica de formulario con lógica de upload.

```typescript
// src/presentation/pages/admin/AdminProductsPage.tsx
// — Solo se muestra el ProductDialog actualizado —

import { useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogFooter,
} from '@/components/ui/dialog';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Button } from '@/components/ui/button';
import { Separator } from '@/components/ui/separator';
import { ImageUploader } from '@/presentation/components/ImageUploader';
import { uploadProductImage } from '@/data/api/images.api';
import { useCatalogStore } from '@/domain/store/catalog.store';
import type { Product } from '@/domain/model/catalog.types';

// Esquema de validación para el formulario del producto (sin imagen, se sube aparte)
const productSchema = z.object({
  name: z.string().min(2, 'El nombre debe tener al menos 2 caracteres'),
  description: z.string().min(10, 'Describe el producto con más detalle'),
  price: z.coerce
    .number({ invalid_type_error: 'Ingresa un precio válido' })
    .positive('El precio debe ser mayor a 0'),
  stock: z.coerce
    .number({ invalid_type_error: 'Ingresa una cantidad válida' })
    .int('El stock debe ser un número entero')
    .min(0, 'El stock no puede ser negativo'),
  category: z.coerce.number({ invalid_type_error: 'Selecciona una categoría' }),
  is_active: z.boolean(),
});

type ProductFormData = z.infer<typeof productSchema>;

interface ProductDialogProps {
  open: boolean;
  product: Product | null; // null = modo creación, Product = modo edición
  onClose: () => void;
  onSaved: (product: Product) => void;
}

export function ProductDialog({
  open,
  product,
  onClose,
  onSaved,
}: ProductDialogProps) {
  const { updateProductInStore } = useCatalogStore();
  const [imageUploadError, setImageUploadError] = useState<string | null>(null);

  const isEditing = product !== null;

  const form = useForm<ProductFormData>({
    resolver: zodResolver(productSchema),
    defaultValues: {
      name: product?.name ?? '',
      description: product?.description ?? '',
      price: product?.price ?? 0,
      stock: product?.stock ?? 0,
      category: product?.category ?? 0,
      is_active: product?.is_active ?? true,
    },
  });

  /**
   * Se ejecuta cuando el usuario confirma la subida de imagen en ImageUploader.
   * Llama directamente a la API y actualiza el store con el producto devuelto.
   */
  async function handleImageUpload(file: File) {
    if (!product) {
      // En modo creación no hay ID de producto aún: la imagen se sube tras guardar.
      // Este botón no se muestra en modo creación (ver condición en el JSX).
      return;
    }

    setImageUploadError(null);
    try {
      const updatedProduct = await uploadProductImage(product.id, file);
      // Actualiza el producto en el store global para que ProductCard y ProductDetailPage
      // reflejen la nueva imagen sin necesidad de recargar la página.
      updateProductInStore(updatedProduct);
      onSaved(updatedProduct);
    } catch (err) {
      const message =
        err instanceof Error ? err.message : 'Error al subir la imagen.';
      setImageUploadError(message);
      // Re-lanza para que ImageUploader muestre su propio mensaje de error.
      throw new Error(message);
    }
  }

  async function onSubmit(data: ProductFormData) {
    // Lógica de creación/edición del producto (igual que módulo 13)
    // Aquí se llama a createProduct o updateProduct de la API correspondiente.
    // Se omite aquí porque no cambia respecto al módulo anterior.
    console.log('Submit product data:', data);
  }

  return (
    <Dialog open={open} onOpenChange={(v) => !v && onClose()}>
      <DialogContent className="max-w-2xl max-h-[90vh] overflow-y-auto">
        <DialogHeader>
          <DialogTitle>
            {isEditing ? `Editar producto: ${product.name}` : 'Nuevo producto'}
          </DialogTitle>
        </DialogHeader>

        <div className="space-y-6 py-2">
          {/* Sección de imagen — solo visible en modo edición */}
          {isEditing && (
            <>
              <div className="space-y-2">
                <h3 className="text-sm font-medium text-foreground">
                  Imagen del producto
                </h3>
                <p className="text-xs text-muted-foreground">
                  La imagen se sube inmediatamente al hacer clic en "Subir
                  imagen". No es necesario guardar el formulario.
                </p>
                <ImageUploader
                  currentImageUrl={product.image}
                  onUpload={handleImageUpload}
                  maxSizeMB={5}
                  className="items-start"
                />
                {imageUploadError && (
                  <p className="text-xs text-destructive">{imageUploadError}</p>
                )}
              </div>
              <Separator />
            </>
          )}

          {/* Formulario de datos del producto */}
          <Form {...form}>
            <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
              <FormField
                control={form.control}
                name="name"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Nombre</FormLabel>
                    <FormControl>
                      <Input placeholder="Nombre del producto" {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <FormField
                control={form.control}
                name="description"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Descripción</FormLabel>
                    <FormControl>
                      <Textarea
                        placeholder="Describe el producto"
                        rows={3}
                        {...field}
                      />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <div className="grid grid-cols-2 gap-4">
                <FormField
                  control={form.control}
                  name="price"
                  render={({ field }) => (
                    <FormItem>
                      <FormLabel>Precio</FormLabel>
                      <FormControl>
                        <Input
                          type="number"
                          step="0.01"
                          min="0"
                          placeholder="0.00"
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
                          placeholder="0"
                          {...field}
                        />
                      </FormControl>
                      <FormMessage />
                    </FormItem>
                  )}
                />
              </div>

              <DialogFooter>
                <Button type="button" variant="outline" onClick={onClose}>
                  Cancelar
                </Button>
                <Button type="submit" disabled={form.formState.isSubmitting}>
                  {form.formState.isSubmitting
                    ? 'Guardando…'
                    : isEditing
                    ? 'Guardar cambios'
                    : 'Crear producto'}
                </Button>
              </DialogFooter>
            </form>
          </Form>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

**Nota sobre modo creación:** en modo creación no existe un `product.id` todavía, por lo que el bloque de `ImageUploader` no se renderiza. El flujo recomendado es: crear el producto primero (obteniendo el ID del backend) y luego subir la imagen en el mismo diálogo reabriendo con el producto recién creado, o redirigir al formulario de edición.

Agrega `updateProductInStore` al Zustand store del catálogo si no existe:

```typescript
// Fragmento de src/domain/store/catalog.store.ts — agregar acción

updateProductInStore: (updatedProduct: Product) => {
  set((state) => ({
    products: state.products.map((p) =>
      p.id === updatedProduct.id ? updatedProduct : p
    ),
  }));
},
```

---

## 14.4 Imagen en ProductCard y ProductDetailPage

### ProductCard

**Archivo:** `src/presentation/components/ProductCard.tsx` (fragmento actualizado)

```typescript
// src/presentation/components/ProductCard.tsx

import { useState } from 'react';
import { ShoppingBag, ShoppingCart } from 'lucide-react';
import { Link } from 'react-router-dom';
import { Button } from '@/components/ui/button';
import { Badge } from '@/components/ui/badge';
import { useCartStore } from '@/domain/store/cart.store';
import type { Product } from '@/domain/model/catalog.types';

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  const addItem = useCartStore((s) => s.addItem);
  // imgError permite mostrar el placeholder si la URL del backend devuelve 404.
  const [imgError, setImgError] = useState(false);

  const showImage = product.image && !imgError;

  return (
    <div className="group flex flex-col rounded-xl border bg-card shadow-sm hover:shadow-md transition-shadow overflow-hidden">
      {/* Contenedor de imagen con relación de aspecto fija 1:1 */}
      <Link
        to={`/catalog/${product.id}`}
        className="block aspect-square overflow-hidden bg-muted"
        aria-label={`Ver detalle de ${product.name}`}
      >
        {showImage ? (
          <img
            src={product.image!}
            alt={product.name}
            loading="lazy"
            className="w-full h-full object-cover group-hover:scale-105 transition-transform duration-300"
            onError={() => setImgError(true)}
          />
        ) : (
          // Placeholder cuando no hay imagen o la URL falla
          <div className="w-full h-full flex items-center justify-center">
            <ShoppingBag className="w-16 h-16 text-muted-foreground/30" />
          </div>
        )}
      </Link>

      {/* Información del producto */}
      <div className="flex flex-col flex-1 p-4 gap-2">
        <div className="flex items-start justify-between gap-2">
          <Link to={`/catalog/${product.id}`}>
            <h3 className="font-semibold text-sm leading-tight line-clamp-2 hover:text-primary transition-colors">
              {product.name}
            </h3>
          </Link>
          <Badge variant="secondary" className="shrink-0 text-xs">
            {product.category_name}
          </Badge>
        </div>

        <p className="text-xs text-muted-foreground line-clamp-2 flex-1">
          {product.description}
        </p>

        <div className="flex items-center justify-between mt-auto pt-2">
          <span className="font-bold text-lg">
            ${Number(product.price).toFixed(2)}
          </span>
          <Button
            size="sm"
            onClick={() => addItem(product)}
            disabled={product.stock === 0}
            aria-label={`Agregar ${product.name} al carrito`}
          >
            <ShoppingCart className="w-4 h-4 mr-1" />
            {product.stock === 0 ? 'Agotado' : 'Agregar'}
          </Button>
        </div>
      </div>
    </div>
  );
}
```

**Por qué `loading="lazy"`:** el atributo nativo de HTML delega al navegador la decisión de cuándo cargar la imagen. Las imágenes fuera del viewport inicial no se descargan hasta que el usuario hace scroll hacia ellas, reduciendo el peso de la carga inicial de la página de catálogo.

**Por qué `object-cover` en `aspect-square`:** garantiza que imágenes de cualquier relación de aspecto (cuadrada, retrato, paisaje) llenen el contenedor sin deformarse, recortando el exceso desde el centro.

### ProductDetailPage

**Archivo:** `src/presentation/pages/catalog/ProductDetailPage.tsx` (fragmento de la sección de imagen)

```typescript
// src/presentation/pages/catalog/ProductDetailPage.tsx
// — Solo se muestra la sección de imagen dentro del componente —

import { useState } from 'react';
import { ShoppingBag } from 'lucide-react';

// Dentro del componente ProductDetailPage, reemplaza la sección de imagen:

function ProductImage({ src, alt }: { src: string | null; alt: string }) {
  const [imgError, setImgError] = useState(false);

  const showImage = src && !imgError;

  return (
    <div className="w-full aspect-square max-w-lg mx-auto lg:mx-0 rounded-2xl overflow-hidden bg-muted border">
      {showImage ? (
        <img
          src={src}
          alt={alt}
          className="w-full h-full object-cover"
          onError={() => setImgError(true)}
        />
      ) : (
        <div className="w-full h-full flex items-center justify-center">
          <ShoppingBag className="w-24 h-24 text-muted-foreground/20" />
        </div>
      )}
    </div>
  );
}

// Uso dentro de ProductDetailPage:
// <ProductImage src={product.image} alt={product.name} />
```

El comportamiento de `onError` es idéntico al de `ProductCard`: si el servidor devuelve un error (imagen eliminada, URL corrupta), el componente pasa a mostrar el placeholder en lugar de una imagen rota.

---

## 14.5 Avatar en ProfilePage

**Archivo:** `src/presentation/pages/profile/ProfilePage.tsx` (fragmento actualizado)

```typescript
// src/presentation/pages/profile/ProfilePage.tsx
// — Se muestran solo las partes nuevas relacionadas con la subida de avatar —

import { ImageUploader } from '@/presentation/components/ImageUploader';
import { uploadUserAvatar } from '@/data/api/images.api';
import { useProfileStore } from '@/domain/store/profile.store';

// Dentro del componente ProfilePage:

function AvatarSection() {
  const { profile, setProfile } = useProfileStore();

  /**
   * Sube el avatar y actualiza el store de perfil localmente.
   * UserAvatar (en el header) lee del mismo store, por lo que se actualiza
   * en toda la app sin necesidad de recargar la página.
   */
  async function handleAvatarUpload(file: File) {
    const updatedProfile = await uploadUserAvatar(file);
    setProfile(updatedProfile);
  }

  return (
    <div className="flex flex-col items-center gap-3 py-4">
      <h2 className="text-sm font-medium text-muted-foreground">
        Foto de perfil
      </h2>
      {/* circular=true aplica rounded-full en el área de previsualización */}
      <ImageUploader
        currentImageUrl={profile?.avatar ?? null}
        onUpload={handleAvatarUpload}
        maxSizeMB={3}
        circular
      />
    </div>
  );
}

// Estructura del componente principal ProfilePage:
// 1. <AvatarSection />          ← nuevo
// 2. <Separator />
// 3. <ProfileForm />            ← ya existía desde módulo 9
```

Asegúrate de que `useProfileStore` exponga la acción `setProfile`:

```typescript
// Fragmento de src/domain/store/profile.store.ts

setProfile: (profile: UserProfile) => set({ profile }),
```

**Flujo de actualización reactiva:** `UserAvatar` en el header ya leía `profile?.avatar` del store en módulos anteriores. Al llamar `setProfile(updatedProfile)`, Zustand notifica a todos los componentes suscritos, incluyendo `UserAvatar`, que se re-renderiza con la nueva URL de avatar sin ninguna acción adicional.

---

## 14.6 ProductDialog completo con imagen integrada

Esta sección muestra la versión completa del diálogo de administración de productos con la imagen integrada al inicio del diálogo. Es la unión de la sección 14.3 con el formulario completo del módulo 13.

```typescript
// src/presentation/pages/admin/AdminProductsPage.tsx — componente completo

import { useEffect, useState } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import {
  Dialog,
  DialogContent,
  DialogHeader,
  DialogTitle,
  DialogFooter,
} from '@/components/ui/dialog';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Button } from '@/components/ui/button';
import { Switch } from '@/components/ui/switch';
import { Separator } from '@/components/ui/separator';
import { ImageUploader } from '@/presentation/components/ImageUploader';
import { uploadProductImage } from '@/data/api/images.api';
import { useCatalogStore } from '@/domain/store/catalog.store';
import type { Product } from '@/domain/model/catalog.types';
import type { Category } from '@/domain/model/catalog.types';

const productSchema = z.object({
  name: z.string().min(2, 'El nombre debe tener al menos 2 caracteres'),
  description: z.string().min(10, 'Describe el producto con más detalle'),
  price: z.coerce.number().positive('El precio debe ser mayor a 0'),
  stock: z.coerce.number().int().min(0, 'El stock no puede ser negativo'),
  category: z.coerce.number({ invalid_type_error: 'Selecciona una categoría' }),
  is_active: z.boolean(),
});

type ProductFormData = z.infer<typeof productSchema>;

interface ProductDialogProps {
  open: boolean;
  product: Product | null;
  categories: Category[];
  onClose: () => void;
  onSave: (data: ProductFormData) => Promise<void>;
}

export function ProductDialog({
  open,
  product,
  categories,
  onClose,
  onSave,
}: ProductDialogProps) {
  const { updateProductInStore } = useCatalogStore();
  const [localProduct, setLocalProduct] = useState<Product | null>(product);

  // Sincroniza localProduct cuando el prop product cambia (al abrir el diálogo).
  useEffect(() => {
    setLocalProduct(product);
  }, [product]);

  const isEditing = localProduct !== null;

  const form = useForm<ProductFormData>({
    resolver: zodResolver(productSchema),
    values: {
      name: localProduct?.name ?? '',
      description: localProduct?.description ?? '',
      price: localProduct?.price ?? 0,
      stock: localProduct?.stock ?? 0,
      category: localProduct?.category ?? (categories[0]?.id ?? 0),
      is_active: localProduct?.is_active ?? true,
    },
  });

  async function handleImageUpload(file: File) {
    if (!localProduct) return;
    const updatedProduct = await uploadProductImage(localProduct.id, file);
    // Actualiza la copia local para refrescar la vista previa en el diálogo.
    setLocalProduct(updatedProduct);
    // Actualiza el store global para ProductCard y ProductDetailPage.
    updateProductInStore(updatedProduct);
  }

  async function onSubmit(data: ProductFormData) {
    await onSave(data);
  }

  return (
    <Dialog open={open} onOpenChange={(v) => !v && onClose()}>
      <DialogContent className="max-w-2xl max-h-[90vh] overflow-y-auto">
        <DialogHeader>
          <DialogTitle>
            {isEditing
              ? `Editar: ${localProduct.name}`
              : 'Nuevo producto'}
          </DialogTitle>
        </DialogHeader>

        <div className="space-y-6 py-2">
          {/* Bloque de imagen — solo en edición */}
          {isEditing && (
            <>
              <div className="space-y-3">
                <div>
                  <h3 className="text-sm font-semibold">Imagen del producto</h3>
                  <p className="text-xs text-muted-foreground mt-0.5">
                    La imagen se sube al instante. No es necesario guardar el
                    formulario primero.
                  </p>
                </div>
                <ImageUploader
                  currentImageUrl={localProduct.image}
                  onUpload={handleImageUpload}
                  maxSizeMB={5}
                />
              </div>
              <Separator />
            </>
          )}

          {/* Formulario de datos */}
          <Form {...form}>
            <form
              id="product-form"
              onSubmit={form.handleSubmit(onSubmit)}
              className="space-y-4"
            >
              <FormField
                control={form.control}
                name="name"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Nombre *</FormLabel>
                    <FormControl>
                      <Input placeholder="Ej. Auriculares Bluetooth" {...field} />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <FormField
                control={form.control}
                name="description"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Descripción *</FormLabel>
                    <FormControl>
                      <Textarea
                        placeholder="Descripción del producto"
                        rows={3}
                        {...field}
                      />
                    </FormControl>
                    <FormMessage />
                  </FormItem>
                )}
              />

              <div className="grid grid-cols-2 gap-4">
                <FormField
                  control={form.control}
                  name="price"
                  render={({ field }) => (
                    <FormItem>
                      <FormLabel>Precio *</FormLabel>
                      <FormControl>
                        <Input
                          type="number"
                          step="0.01"
                          min="0"
                          placeholder="0.00"
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
                      <FormLabel>Stock *</FormLabel>
                      <FormControl>
                        <Input
                          type="number"
                          min="0"
                          placeholder="0"
                          {...field}
                        />
                      </FormControl>
                      <FormMessage />
                    </FormItem>
                  )}
                />
              </div>

              <FormField
                control={form.control}
                name="category"
                render={({ field }) => (
                  <FormItem>
                    <FormLabel>Categoría *</FormLabel>
                    <Select
                      onValueChange={field.onChange}
                      value={String(field.value)}
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

              <FormField
                control={form.control}
                name="is_active"
                render={({ field }) => (
                  <FormItem className="flex items-center justify-between rounded-lg border p-4">
                    <div>
                      <FormLabel className="text-base">
                        Producto activo
                      </FormLabel>
                      <p className="text-xs text-muted-foreground">
                        Los productos inactivos no aparecen en el catálogo.
                      </p>
                    </div>
                    <FormControl>
                      <Switch
                        checked={field.value}
                        onCheckedChange={field.onChange}
                      />
                    </FormControl>
                  </FormItem>
                )}
              />
            </form>
          </Form>
        </div>

        <DialogFooter>
          <Button type="button" variant="outline" onClick={onClose}>
            Cancelar
          </Button>
          <Button
            type="submit"
            form="product-form"
            disabled={form.formState.isSubmitting}
          >
            {form.formState.isSubmitting
              ? 'Guardando…'
              : isEditing
              ? 'Guardar cambios'
              : 'Crear producto'}
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

**Técnica `form="product-form"`:** el `<Button type="submit">` se encuentra fuera del `<form>` (dentro de `DialogFooter`). Usar el atributo `form` con el `id` del formulario asocia el botón al formulario sin necesidad de mover el botón dentro del árbol del formulario, manteniendo la estructura semántica del diálogo.

---

## 14.7 Checkpoint y tabla de resumen

### Verificación paso a paso

**1. Verifica la subida de imagen de producto:**
```bash
# En DevTools Network, filtra por "products"
# Al subir una imagen en el diálogo admin, deberías ver:
# PATCH /api/products/{id}/   → 200 OK
# Content-Type: multipart/form-data; boundary=...
```

**2. Verifica que la imagen aparece en el catálogo:**
- Navega a `/catalog` — el producto debe mostrar la imagen subida.
- Navega a `/catalog/{id}` — la imagen debe aparecer en tamaño grande.
- Recarga la página — la imagen persiste (viene del backend).

**3. Verifica la subida de avatar:**
- Navega a `/profile`.
- Haz clic en el círculo de avatar, selecciona una imagen, haz clic en "Subir imagen".
- El `UserAvatar` en el header debe actualizarse inmediatamente.

**4. Verifica el manejo de errores:**
- Intenta subir un archivo de más de 5 MB — debe aparecer el mensaje de error antes de llamar a la API.
- Desconecta el backend y prueba la subida — debe aparecer el toast de error.

### Tabla de resumen del módulo 14

| Sección | Archivo | Descripción |
|---------|---------|-------------|
| 14.1 | `src/data/api/images.api.ts` | Funciones `uploadProductImage` y `uploadUserAvatar` con FormData |
| 14.2 | `src/presentation/components/ImageUploader.tsx` | Componente reutilizable con vista previa, validación de tamaño y estados |
| 14.3 | `src/presentation/pages/admin/AdminProductsPage.tsx` | Integración de ImageUploader en ProductDialog (admin) |
| 14.4 | `src/presentation/components/ProductCard.tsx` | Imagen con lazy loading, aspect-square, fallback con onError |
| 14.4 | `src/presentation/pages/catalog/ProductDetailPage.tsx` | Imagen grande con fallback en página de detalle |
| 14.5 | `src/presentation/pages/profile/ProfilePage.tsx` | Avatar circular con subida y actualización reactiva del store |
| 14.6 | `src/presentation/pages/admin/AdminProductsPage.tsx` | ProductDialog completo con imagen al inicio, form con id externo |

### Conceptos clave

| Concepto | Descripción |
|----------|-------------|
| `FormData` + `PATCH` | Forma estándar de enviar archivos a APIs REST; Axios fija `Content-Type: multipart/form-data` automáticamente |
| `URL.createObjectURL` | Genera una URL blob local para vista previa instantánea sin subir al servidor |
| `URL.revokeObjectURL` | Libera la memoria del blob cuando ya no se necesita |
| `loading="lazy"` | Atributo HTML nativo que difiere la carga de imágenes fuera del viewport |
| `onError` en `<img>` | Permite detectar URLs rotas y mostrar un placeholder como fallback |
| `circular` prop | Parámetro de configuración que adapta el componente para avatares sin duplicar lógica |
| `form="id"` | Asocia un botón submit externo al formulario sin romper la estructura semántica |
