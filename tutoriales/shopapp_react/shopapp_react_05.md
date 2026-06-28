# ShopApp React Web — Módulo 5

## Búsqueda y filtros — SearchBar, filtros por categoría, precio y ordenamiento

**Duración estimada:** 2–3 horas
**Prerequisitos:** Módulos 1–4 completados. El catálogo muestra productos con paginación.

---

> **Objetivo**
> Enriquecer el catálogo con búsqueda por texto con debounce, selector de ordenamiento,
> panel de filtros responsivo (sidebar en escritorio, drawer en móvil) y la vista parcial
> de detalle de producto que se completará en el módulo 6.
>
> **Checkpoint final**
> - Escribir en el SearchBar filtra los productos después de 300 ms sin hacer llamadas en cada tecla.
> - El botón X limpia la búsqueda instantáneamente.
> - El selector de ordenamiento reordena los resultados al cambiar.
> - En pantallas `lg` se ve el sidebar de filtros; en móvil aparece un drawer al pulsar el botón Filtros.
> - La ruta `/products/:id` muestra imagen, nombre, precio y descripción del producto.
> - `npm run build` compila sin errores de TypeScript.

---

## 5.1 Actualización del CatalogStore

**Archivo:** `src/domain/store/catalog.store.ts` (versión actualizada)

Se añade `activeFilterCount` para contar cuántos filtros no predeterminados están activos. Este valor lo usará el botón de filtros en móvil para mostrar una insignia numérica.

```typescript
// src/domain/store/catalog.store.ts

import { create } from 'zustand';
import { getCategories, getProducts } from '../../data/api/catalog.api';
import type { Category, CatalogFilters, Product } from '../model/catalog.types';

interface CatalogState {
  products: Product[];
  categories: Category[];
  filters: CatalogFilters;
  isLoading: boolean;
  error: string | null;
  totalCount: number;
  currentPage: number;
  activeFilterCount: number;

  fetchProducts: () => Promise<void>;
  fetchCategories: () => Promise<void>;
  setFilters: (partial: Partial<CatalogFilters>) => void;
  resetFilters: () => void;
  setPage: (page: number) => void;
}

const DEFAULT_FILTERS: CatalogFilters = {
  search: '',
  categoryId: null,
  ordering: 'name',
};

function countActiveFilters(filters: CatalogFilters): number {
  let count = 0;
  if (filters.search.trim() !== '') count++;
  if (filters.categoryId !== null) count++;
  if (filters.ordering !== DEFAULT_FILTERS.ordering) count++;
  return count;
}

export const useCatalogStore = create<CatalogState>((set, get) => ({
  products: [],
  categories: [],
  filters: { ...DEFAULT_FILTERS },
  isLoading: false,
  error: null,
  totalCount: 0,
  currentPage: 1,
  activeFilterCount: 0,

  fetchProducts: async () => {
    set({ isLoading: true, error: null });
    try {
      const { filters, currentPage } = get();
      const data = await getProducts(filters, currentPage);
      set({
        products: data.results,
        totalCount: data.count,
        isLoading: false,
      });
    } catch (err) {
      const message = err instanceof Error ? err.message : 'Error al cargar productos';
      set({ error: message, isLoading: false });
    }
  },

  fetchCategories: async () => {
    try {
      const data = await getCategories();
      set({ categories: data });
    } catch {
      // Las categorías son opcionales; no bloquear la UI si fallan
    }
  },

  setFilters: (partial) => {
    set((state) => {
      const newFilters = { ...state.filters, ...partial };
      return {
        filters: newFilters,
        currentPage: 1,
        activeFilterCount: countActiveFilters(newFilters),
      };
    });
  },

  resetFilters: () => {
    set({
      filters: { ...DEFAULT_FILTERS },
      currentPage: 1,
      activeFilterCount: 0,
    });
  },

  setPage: (page) => {
    set({ currentPage: page });
  },
}));
```

`countActiveFilters` es una función pura que no vive en el store, facilitando su prueba unitaria. `setFilters` la invoca cada vez que los filtros cambian para mantener `activeFilterCount` siempre sincronizado.

---

## 5.2 Componente SearchBar

**Archivo:** `src/presentation/components/SearchBar.tsx`

El debounce se implementa con `useEffect` y `setTimeout` sin ninguna librería externa, manteniendo las dependencias del proyecto al mínimo. El valor del input es controlado localmente para dar respuesta inmediata al usuario mientras el store se actualiza tras la pausa.

```typescript
// src/presentation/components/SearchBar.tsx

import { useEffect, useRef, useState } from 'react';
import { Search, X } from 'lucide-react';
import { Input } from '@/components/ui/input';
import { Button } from '@/components/ui/button';
import { useCatalogStore } from '../../domain/store/catalog.store';

const DEBOUNCE_MS = 300;

export function SearchBar() {
  const storeSearch = useCatalogStore((s) => s.filters.search);
  const setFilters = useCatalogStore((s) => s.setFilters);
  const fetchProducts = useCatalogStore((s) => s.fetchProducts);

  const [localValue, setLocalValue] = useState(storeSearch);
  const timerRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  // Sincronizar el valor local si el store se resetea externamente
  useEffect(() => {
    setLocalValue(storeSearch);
  }, [storeSearch]);

  const handleChange = (value: string) => {
    setLocalValue(value);

    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }

    timerRef.current = setTimeout(() => {
      setFilters({ search: value });
      fetchProducts();
    }, DEBOUNCE_MS);
  };

  const handleClear = () => {
    if (timerRef.current) {
      clearTimeout(timerRef.current);
    }
    setLocalValue('');
    setFilters({ search: '' });
    fetchProducts();
  };

  // Limpiar el timer al desmontar el componente
  useEffect(() => {
    return () => {
      if (timerRef.current) {
        clearTimeout(timerRef.current);
      }
    };
  }, []);

  return (
    <div className="relative w-full max-w-sm">
      <Search className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
      <Input
        type="text"
        placeholder="Buscar productos..."
        value={localValue}
        onChange={(e) => handleChange(e.target.value)}
        className="pl-9 pr-9"
        aria-label="Buscar productos"
      />
      {localValue && (
        <Button
          variant="ghost"
          size="icon"
          className="absolute right-1 top-1/2 h-7 w-7 -translate-y-1/2"
          onClick={handleClear}
          aria-label="Limpiar búsqueda"
        >
          <X className="h-4 w-4" />
        </Button>
      )}
    </div>
  );
}
```

`useRef` almacena el timer para que su valor persista entre renders sin provocar re-renders adicionales. El botón X llama directamente a `fetchProducts` sin esperar el debounce para que el borrado se aplique de inmediato.

---

## 5.3 Componente SortSelect

**Archivo:** `src/presentation/components/SortSelect.tsx`

`SortSelect` usa el componente `Select` de shadcn/ui para garantizar accesibilidad y consistencia visual. Las etiquetas muestran texto legible al usuario mientras los valores corresponden exactamente a los parámetros que espera el backend de Django REST Framework.

```typescript
// src/presentation/components/SortSelect.tsx

import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from '@/components/ui/select';
import { useCatalogStore } from '../../domain/store/catalog.store';
import type { OrderingOption } from '../../domain/model/catalog.types';

interface SortOption {
  label: string;
  value: OrderingOption;
}

const SORT_OPTIONS: SortOption[] = [
  { label: 'Nombre A-Z', value: 'name' },
  { label: 'Nombre Z-A', value: '-name' },
  { label: 'Precio ↑', value: 'price' },
  { label: 'Precio ↓', value: '-price' },
  { label: 'Stock ↑', value: 'stock' },
];

export function SortSelect() {
  const ordering = useCatalogStore((s) => s.filters.ordering);
  const setFilters = useCatalogStore((s) => s.setFilters);
  const fetchProducts = useCatalogStore((s) => s.fetchProducts);

  const handleChange = (value: string) => {
    setFilters({ ordering: value as OrderingOption });
    fetchProducts();
  };

  return (
    <div className="flex items-center gap-2">
      <span className="shrink-0 text-sm text-muted-foreground">Ordenar por</span>
      <Select value={ordering} onValueChange={handleChange}>
        <SelectTrigger className="w-40">
          <SelectValue />
        </SelectTrigger>
        <SelectContent>
          {SORT_OPTIONS.map((opt) => (
            <SelectItem key={opt.value} value={opt.value}>
              {opt.label}
            </SelectItem>
          ))}
        </SelectContent>
      </Select>
    </div>
  );
}
```

Definir `SORT_OPTIONS` como una constante fuera del componente evita recrear el array en cada render. El cast `value as OrderingOption` es seguro porque los valores del `SelectItem` están definidos con el mismo tipo.

---

## 5.4 Componente FilterPanel

**Archivo:** `src/presentation/components/FilterPanel.tsx`

`FilterPanel` combina los filtros de categoría y ordenamiento en un contenedor único. En escritorio funciona como un `aside` estático; en móvil se convierte en un drawer usando el componente `Sheet` de shadcn/ui, manteniendo el mismo contenido interno sin duplicar código.

```typescript
// src/presentation/components/FilterPanel.tsx

import { SlidersHorizontal } from 'lucide-react';
import { Button } from '@/components/ui/button';
import {
  Sheet,
  SheetContent,
  SheetHeader,
  SheetTitle,
  SheetTrigger,
} from '@/components/ui/sheet';
import { Badge } from '@/components/ui/badge';
import { CategoryFilter } from './CategoryFilter';
import { SortSelect } from './SortSelect';
import { useCatalogStore } from '../../domain/store/catalog.store';

function FilterContent() {
  const resetFilters = useCatalogStore((s) => s.resetFilters);
  const fetchProducts = useCatalogStore((s) => s.fetchProducts);
  const activeFilterCount = useCatalogStore((s) => s.activeFilterCount);

  const handleReset = () => {
    resetFilters();
    fetchProducts();
  };

  return (
    <div className="flex flex-col gap-6">
      <div>
        <p className="mb-2 text-sm font-medium">Categoría</p>
        <div className="flex flex-col gap-1">
          <CategoryFilter layout="vertical" />
        </div>
      </div>

      <div>
        <p className="mb-2 text-sm font-medium">Ordenamiento</p>
        <SortSelect />
      </div>

      {activeFilterCount > 0 && (
        <Button variant="ghost" size="sm" onClick={handleReset} className="self-start">
          Limpiar filtros
        </Button>
      )}
    </div>
  );
}

interface FilterPanelProps {
  className?: string;
}

export function FilterPanel({ className }: FilterPanelProps) {
  const activeFilterCount = useCatalogStore((s) => s.activeFilterCount);

  return (
    <>
      {/* Sidebar visible en escritorio */}
      <aside className={`hidden lg:block ${className ?? ''}`}>
        <FilterContent />
      </aside>

      {/* Botón + Sheet visible en móvil */}
      <div className="lg:hidden">
        <Sheet>
          <SheetTrigger asChild>
            <Button variant="outline" size="sm" className="relative">
              <SlidersHorizontal className="mr-2 h-4 w-4" />
              Filtros
              {activeFilterCount > 0 && (
                <Badge className="absolute -right-2 -top-2 flex h-5 w-5 items-center justify-center rounded-full p-0 text-xs">
                  {activeFilterCount}
                </Badge>
              )}
            </Button>
          </SheetTrigger>
          <SheetContent side="left" className="w-72">
            <SheetHeader className="mb-6">
              <SheetTitle>Filtros</SheetTitle>
            </SheetHeader>
            <FilterContent />
          </SheetContent>
        </Sheet>
      </div>
    </>
  );
}
```

`FilterContent` es un componente interno que encapsula el contenido del panel para reutilizarlo tanto en el `aside` de escritorio como dentro del `Sheet` móvil sin duplicar JSX. La insignia numérica usa posicionamiento absoluto sobre el botón para no alterar el flujo del layout.

---

## 5.5 Actualización de CategoryFilter para layout vertical

**Archivo:** `src/presentation/components/CategoryFilter.tsx` (versión actualizada)

Se añade la prop `layout` para que el mismo componente funcione tanto en la fila horizontal del encabezado como en la lista vertical del sidebar, sin condicionales dispersos en los componentes padre.

```typescript
// src/presentation/components/CategoryFilter.tsx

import { useCatalogStore } from '../../domain/store/catalog.store';
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils';

interface CategoryFilterProps {
  layout?: 'horizontal' | 'vertical';
}

export function CategoryFilter({ layout = 'horizontal' }: CategoryFilterProps) {
  const categories = useCatalogStore((s) => s.categories);
  const filters = useCatalogStore((s) => s.filters);
  const setFilters = useCatalogStore((s) => s.setFilters);
  const fetchProducts = useCatalogStore((s) => s.fetchProducts);

  const handleSelect = (categoryId: number | null) => {
    setFilters({ categoryId });
    fetchProducts();
  };

  const containerClass =
    layout === 'vertical'
      ? 'flex flex-col gap-1'
      : 'flex gap-2 overflow-x-auto pb-2 scrollbar-thin';

  const buttonClass = layout === 'vertical' ? 'justify-start' : 'shrink-0';

  return (
    <div className={containerClass}>
      <Button
        variant={filters.categoryId === null ? 'default' : 'ghost'}
        size="sm"
        className={cn(buttonClass, filters.categoryId === null && 'pointer-events-none')}
        onClick={() => handleSelect(null)}
      >
        Todos
      </Button>

      {categories.map((cat) => (
        <Button
          key={cat.id}
          variant={filters.categoryId === cat.id ? 'default' : 'ghost'}
          size="sm"
          className={cn(
            buttonClass,
            filters.categoryId === cat.id && 'pointer-events-none',
          )}
          onClick={() => handleSelect(cat.id)}
        >
          {cat.name}
          {cat.product_count !== undefined && (
            <span className="ml-auto text-xs opacity-60">({cat.product_count})</span>
          )}
        </Button>
      ))}
    </div>
  );
}
```

La prop `layout` controla las clases del contenedor y los botones mediante variables locales, evitando condicionales en línea que dificultan la lectura. El `ml-auto` en el contador mueve el numero al extremo derecho en el layout vertical, imitando el patrón de menús de navegacion.

---

## 5.6 CatalogPage actualizada

**Archivo:** `src/presentation/pages/catalog/CatalogPage.tsx` (versión completa del módulo 5)

La página adopta un layout de dos columnas en escritorio (`sidebar + grilla`) con el `FilterPanel` integrado. En móvil la columna del sidebar desaparece y el botón de filtros queda en la barra de herramientas.

```typescript
// src/presentation/pages/catalog/CatalogPage.tsx

import { useEffect } from 'react';
import { AppShell } from '../../components/AppShell';
import { CategoryFilter } from '../../components/CategoryFilter';
import { FilterPanel } from '../../components/FilterPanel';
import { ProductCard } from '../../components/ProductCard';
import { ProductCardSkeleton } from '../../components/ProductCardSkeleton';
import { SearchBar } from '../../components/SearchBar';
import { useCatalogStore } from '../../../domain/store/catalog.store';
import { Button } from '@/components/ui/button';
import { ChevronLeft, ChevronRight } from 'lucide-react';

const PAGE_SIZE = 12;

export function CatalogPage() {
  const products = useCatalogStore((s) => s.products);
  const categories = useCatalogStore((s) => s.categories);
  const isLoading = useCatalogStore((s) => s.isLoading);
  const totalCount = useCatalogStore((s) => s.totalCount);
  const currentPage = useCatalogStore((s) => s.currentPage);
  const fetchProducts = useCatalogStore((s) => s.fetchProducts);
  const fetchCategories = useCatalogStore((s) => s.fetchCategories);
  const setPage = useCatalogStore((s) => s.setPage);

  const totalPages = Math.max(1, Math.ceil(totalCount / PAGE_SIZE));

  useEffect(() => {
    if (categories.length === 0) {
      fetchCategories();
    }
  }, []);

  useEffect(() => {
    fetchProducts();
  }, [currentPage]);

  const handlePrev = () => {
    if (currentPage > 1) setPage(currentPage - 1);
  };

  const handleNext = () => {
    if (currentPage < totalPages) setPage(currentPage + 1);
  };

  return (
    <AppShell>
      <div className="container mx-auto px-4 py-6">
        <h1 className="mb-4 text-2xl font-bold">Catálogo</h1>

        {/* Barra de herramientas */}
        <div className="mb-4 flex flex-wrap items-center gap-3">
          <SearchBar />
          {/* Botón de filtros solo visible en móvil */}
          <div className="lg:hidden">
            <FilterPanel />
          </div>
          <p className="ml-auto text-sm text-muted-foreground">
            {totalCount} resultado{totalCount !== 1 ? 's' : ''}
          </p>
        </div>

        {/* Filtro horizontal de categorías solo en escritorio */}
        <div className="mb-6 hidden lg:block">
          <CategoryFilter layout="horizontal" />
        </div>

        {/* Layout de dos columnas en escritorio */}
        <div className="lg:grid lg:grid-cols-[240px_1fr] lg:gap-8">
          {/* Sidebar de filtros (escritorio) */}
          <FilterPanel className="sticky top-20 h-fit" />

          {/* Columna principal */}
          <div>
            {isLoading ? (
              <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 md:grid-cols-3">
                {Array.from({ length: 8 }).map((_, i) => (
                  <ProductCardSkeleton key={i} />
                ))}
              </div>
            ) : products.length === 0 ? (
              <div className="flex min-h-64 items-center justify-center">
                <p className="text-muted-foreground">No se encontraron productos</p>
              </div>
            ) : (
              <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 md:grid-cols-3">
                {products.map((product) => (
                  <ProductCard key={product.id} product={product} />
                ))}
              </div>
            )}

            {/* Paginación */}
            {!isLoading && products.length > 0 && (
              <div className="mt-8 flex items-center justify-center gap-4">
                <Button
                  variant="outline"
                  size="sm"
                  onClick={handlePrev}
                  disabled={currentPage === 1}
                >
                  <ChevronLeft className="mr-1 h-4 w-4" />
                  Anterior
                </Button>
                <span className="text-sm text-muted-foreground">
                  Página {currentPage} de {totalPages}
                </span>
                <Button
                  variant="outline"
                  size="sm"
                  onClick={handleNext}
                  disabled={currentPage === totalPages}
                >
                  Siguiente
                  <ChevronRight className="ml-1 h-4 w-4" />
                </Button>
              </div>
            )}
          </div>
        </div>
      </div>
    </AppShell>
  );
}
```

La clase `lg:grid-cols-[240px_1fr]` usa una columna de ancho fijo para el sidebar y `1fr` para la grilla de productos, lo que mantiene el sidebar estable mientras los productos se redistribuyen. La visibilidad del `FilterPanel` se controla con clases `hidden lg:block` para evitar renderizar el Sheet en escritorio innecesariamente.

---

## 5.7 ProductDetailPage (vista parcial)

**Archivo:** `src/presentation/pages/catalog/ProductDetailPage.tsx`

Esta vista carga y muestra un producto individual. El botón "Agregar al carrito" queda deshabilitado hasta que el módulo 6 implemente `CartStore`. El skeleton mantiene la estructura de la vista final para evitar saltos de layout cuando los datos cargan.

```typescript
// src/presentation/pages/catalog/ProductDetailPage.tsx

import { useEffect, useState } from 'react';
import { useNavigate, useParams } from 'react-router-dom';
import { ArrowLeft, ShoppingBag, ShoppingCart } from 'lucide-react';
import { AppShell } from '../../components/AppShell';
import { getProductById } from '../../../data/api/catalog.api';
import type { Product } from '../../../domain/model/catalog.types';
import { formatPrice } from '../../../core/utils/formatters';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Skeleton } from '@/components/ui/skeleton';

export function ProductDetailPage() {
  const { id } = useParams<{ id: string }>();
  const navigate = useNavigate();

  const [product, setProduct] = useState<Product | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!id) return;

    setIsLoading(true);
    setError(null);

    getProductById(Number(id))
      .then((data) => setProduct(data))
      .catch(() => setError('No se pudo cargar el producto.'))
      .finally(() => setIsLoading(false));
  }, [id]);

  return (
    <AppShell>
      <div className="container mx-auto max-w-4xl px-4 py-6">
        <Button
          variant="ghost"
          size="sm"
          className="mb-6"
          onClick={() => navigate(-1)}
        >
          <ArrowLeft className="mr-2 h-4 w-4" />
          Volver
        </Button>

        {isLoading && <ProductDetailSkeleton />}

        {error && (
          <div className="flex min-h-64 items-center justify-center">
            <p className="text-destructive">{error}</p>
          </div>
        )}

        {!isLoading && !error && product && (
          <div className="grid gap-8 md:grid-cols-2">
            {/* Imagen */}
            <div className="overflow-hidden rounded-lg">
              {product.image ? (
                <img
                  src={product.image}
                  alt={product.name}
                  className="h-full w-full object-cover"
                />
              ) : (
                <div className="flex h-72 items-center justify-center rounded-lg bg-muted">
                  <ShoppingBag className="h-20 w-20 text-muted-foreground" />
                </div>
              )}
            </div>

            {/* Información */}
            <div className="flex flex-col gap-4">
              <div>
                <Badge variant="secondary" className="mb-2">
                  {product.category_name}
                </Badge>
                <h1 className="text-2xl font-bold">{product.name}</h1>
              </div>

              <p className="text-3xl font-bold text-primary">
                {formatPrice(product.price)}
              </p>

              {product.stock > 0 ? (
                <Badge
                  variant="outline"
                  className="w-fit border-green-500 text-green-600"
                >
                  {product.stock} unidades disponibles
                </Badge>
              ) : (
                <Badge variant="destructive" className="w-fit">
                  Agotado
                </Badge>
              )}

              <p className="text-sm leading-relaxed text-muted-foreground">
                {product.description}
              </p>

              <Button
                size="lg"
                className="mt-auto"
                disabled={product.stock === 0}
                title="Disponible en el módulo 6"
              >
                <ShoppingCart className="mr-2 h-5 w-5" />
                Agregar al carrito
              </Button>
            </div>
          </div>
        )}
      </div>
    </AppShell>
  );
}

function ProductDetailSkeleton() {
  return (
    <div className="grid gap-8 md:grid-cols-2">
      <Skeleton className="h-72 w-full rounded-lg" />
      <div className="flex flex-col gap-4">
        <Skeleton className="h-5 w-24 rounded-full" />
        <Skeleton className="h-8 w-3/4" />
        <Skeleton className="h-9 w-32" />
        <Skeleton className="h-5 w-28 rounded-full" />
        <div className="space-y-2">
          <Skeleton className="h-4 w-full" />
          <Skeleton className="h-4 w-full" />
          <Skeleton className="h-4 w-2/3" />
        </div>
        <Skeleton className="mt-auto h-11 w-full" />
      </div>
    </div>
  );
}
```

`ProductDetailSkeleton` se define en el mismo archivo como función privada porque solo la usa `ProductDetailPage`. El botón queda habilitado solo cuando hay stock y el título `"Disponible en el módulo 6"` sirve como recordatorio durante el desarrollo.

---

## 5.8 Registro de la ruta en el router

Añade `ProductDetailPage` al router y reemplaza el placeholder del módulo 4:

```typescript
// src/presentation/router/AppRouter.tsx  (fragmento — reemplazar la línea del módulo 4)

import { CatalogPage } from '../pages/catalog/CatalogPage';
import { ProductDetailPage } from '../pages/catalog/ProductDetailPage';

// Dentro de <Routes>:
<Route path="/catalog" element={<CatalogPage />} />
<Route path="/products/:id" element={<ProductDetailPage />} />
```

Con esta ruta, el botón "Ver detalle" de `ProductCard` navegará correctamente a `/products/1`, `/products/2`, etc.

---

## 5.9 Checkpoint y tabla resumen

Antes de continuar con el módulo 6, verifica:

- [ ] Escribir en el SearchBar y pausar 300 ms dispara una nueva búsqueda (visible en la pestaña Network de DevTools).
- [ ] Pulsar el botón X borra el texto y recarga todos los productos sin esperar el debounce.
- [ ] El selector de ordenamiento cambia el orden de los productos en la grilla.
- [ ] En una pantalla `< 1024 px`, el sidebar no es visible y el botón "Filtros" abre un drawer lateral.
- [ ] La insignia numérica sobre el botón "Filtros" muestra cuántos filtros están activos.
- [ ] "Limpiar filtros" restablece búsqueda, categoría y ordenamiento a sus valores por defecto.
- [ ] `/products/1` muestra la imagen, nombre, precio y descripción del producto con id 1.
- [ ] El botón "Volver" en ProductDetailPage regresa a la página anterior del historial.
- [ ] `npm run build` compila sin errores.

---

### Tabla resumen del módulo 5

| Archivo | Capa | Responsabilidad |
|---|---|---|
| `src/domain/store/catalog.store.ts` | Domain / Store | Agrega `activeFilterCount` y garantiza reset de página en `setFilters` |
| `src/presentation/components/SearchBar.tsx` | Presentation | Input con debounce de 300 ms, botón de limpieza y sincronización con el store |
| `src/presentation/components/SortSelect.tsx` | Presentation | Select de shadcn con las 5 opciones de ordenamiento del backend |
| `src/presentation/components/CategoryFilter.tsx` | Presentation | Botones de categoría con soporte de layout horizontal y vertical |
| `src/presentation/components/FilterPanel.tsx` | Presentation | Panel de filtros: aside en escritorio, Sheet drawer en móvil |
| `src/presentation/pages/catalog/CatalogPage.tsx` | Presentation | Página actualizada con SearchBar, FilterPanel y layout de dos columnas |
| `src/presentation/pages/catalog/ProductDetailPage.tsx` | Presentation | Vista de detalle de producto con skeleton, imagen y botón de carrito (parcial) |
