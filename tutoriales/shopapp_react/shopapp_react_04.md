# ShopApp React Web — Módulo 4

## Catálogo — Listado de productos, categorías y CatalogStore

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulos 1–3 completados. El backend expone `/api/categories/` y `/api/products/`.

---

> **Objetivo**
> Implementar la capa completa de catálogo: tipos de dominio, cliente de API, Zustand store, y todas
> las vistas y componentes necesarios para listar productos con categorías, paginación y estados de
> carga y vacío.
>
> **Checkpoint final**
> - La ruta `/catalog` muestra una grilla de productos reales del backend.
> - El filtro por categoría funciona sin recargar la página.
> - La paginación prev/next navega correctamente entre páginas.
> - El estado de carga muestra tarjetas skeleton y el estado vacío muestra un mensaje centrado.
> - `npm run build` compila sin errores de TypeScript.

---

## 4.1 Tipos de dominio del catálogo

**Archivo:** `src/domain/model/catalog.types.ts`

Este archivo define todas las interfaces TypeScript que representan los recursos del catálogo. Centralizar los tipos en la capa `domain/model` garantiza que tanto la capa de datos como la de presentación compartan el mismo contrato, evitando duplicación y errores de forma silenciosa.

```typescript
// src/domain/model/catalog.types.ts

export interface Category {
  id: number;
  name: string;
  description: string;
  product_count?: number;
}

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

export type OrderingOption = 'price' | '-price' | 'name' | '-name' | 'stock';

export interface CatalogFilters {
  search: string;
  categoryId: number | null;
  ordering: OrderingOption;
}

export interface PaginatedResponse<T> {
  count: number;
  next: string | null;
  previous: string | null;
  results: T[];
}
```

`Category` y `Product` mapean directamente los serializadores Django. `CatalogFilters` encapsula el estado de búsqueda, categoría y ordenamiento que el store usará para construir los query params. `PaginatedResponse<T>` es un genérico reutilizable para cualquier endpoint paginado de DRF.

---

## 4.2 API del catálogo

**Archivo:** `src/data/api/catalog.api.ts`

La capa de API es la única que conoce las URLs del backend. Todas las funciones reciben parámetros tipados y devuelven promesas con los tipos del dominio, aislando el resto de la app de los detalles de HTTP.

```typescript
// src/data/api/catalog.api.ts

import { apiClient } from './api.client';
import type { Category, CatalogFilters, PaginatedResponse, Product } from '../../domain/model/catalog.types';

export async function getCategories(): Promise<Category[]> {
  const response = await apiClient.get<Category[]>('/categories/');
  return response.data;
}

export async function getProducts(
  filters?: Partial<CatalogFilters>,
  page: number = 1,
): Promise<PaginatedResponse<Product>> {
  const params: Record<string, string | number> = {
    page,
    page_size: 12,
  };

  if (filters?.search) {
    params.search = filters.search;
  }
  if (filters?.categoryId !== undefined && filters.categoryId !== null) {
    params.category = filters.categoryId;
  }
  if (filters?.ordering) {
    params.ordering = filters.ordering;
  }

  const response = await apiClient.get<PaginatedResponse<Product>>('/products/', { params });
  return response.data;
}

export async function getProductById(id: number): Promise<Product> {
  const response = await apiClient.get<Product>(`/products/${id}/`);
  return response.data;
}
```

`getProducts` construye los query params solo con los filtros que tienen valor real, evitando enviar `category=null` o `search=` al backend. Los valores por defecto de `page` y `page_size` se centralizan aquí para no duplicarlos en el store.

---

## 4.3 CatalogStore con Zustand

**Archivo:** `src/domain/store/catalog.store.ts`

El store es el único origen de verdad del estado del catálogo. Contiene los datos (productos, categorías, filtros, paginación) y las acciones que los modifican, manteniendo la lógica fuera de los componentes.

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

export const useCatalogStore = create<CatalogState>((set, get) => ({
  products: [],
  categories: [],
  filters: { ...DEFAULT_FILTERS },
  isLoading: false,
  error: null,
  totalCount: 0,
  currentPage: 1,

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
    set((state) => ({
      filters: { ...state.filters, ...partial },
      currentPage: 1,
    }));
  },

  resetFilters: () => {
    set({ filters: { ...DEFAULT_FILTERS }, currentPage: 1 });
  },

  setPage: (page) => {
    set({ currentPage: page });
  },
}));
```

`setFilters` siempre resetea `currentPage` a 1 para que un cambio de filtro no deje al usuario en una página inexistente. `fetchCategories` captura el error en silencio porque la app debe seguir funcionando aunque las categorías no carguen.

---

## 4.4 Componente ProductCard

**Archivo:** `src/presentation/components/ProductCard.tsx`

`ProductCard` es un componente de presentación puro: recibe un `Product` y renderiza la tarjeta sin conocer el store ni la API. Esto facilita su reutilización y pruebas unitarias aisladas.

```typescript
// src/presentation/components/ProductCard.tsx

import { ShoppingBag } from 'lucide-react';
import { Link } from 'react-router-dom';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Card, CardContent, CardFooter, CardHeader } from '@/components/ui/card';
import { formatPrice } from '../../core/utils/formatters';
import type { Product } from '../../domain/model/catalog.types';

interface ProductCardProps {
  product: Product;
}

export function ProductCard({ product }: ProductCardProps) {
  const { id, name, category_name, price, stock, image } = product;

  return (
    <Card className="flex flex-col overflow-hidden transition-shadow hover:shadow-md">
      <CardHeader className="p-0">
        {image ? (
          <img
            src={image}
            alt={name}
            className="h-48 w-full object-cover"
          />
        ) : (
          <div className="flex h-48 w-full items-center justify-center bg-muted">
            <ShoppingBag className="h-12 w-12 text-muted-foreground" />
          </div>
        )}
      </CardHeader>

      <CardContent className="flex flex-1 flex-col gap-2 p-4">
        <div className="flex items-start justify-between gap-2">
          <h3 className="line-clamp-2 text-sm font-semibold leading-tight">{name}</h3>
          <Badge variant="secondary" className="shrink-0 text-xs">
            {category_name}
          </Badge>
        </div>

        <p className="text-lg font-bold text-primary">{formatPrice(price)}</p>

        {stock > 0 ? (
          <Badge variant="outline" className="w-fit border-green-500 text-green-600">
            {stock} en stock
          </Badge>
        ) : (
          <Badge variant="destructive" className="w-fit">
            Agotado
          </Badge>
        )}
      </CardContent>

      <CardFooter className="p-4 pt-0">
        <Button asChild className="w-full" size="sm">
          <Link to={`/products/${id}`}>Ver detalle</Link>
        </Button>
      </CardFooter>
    </Card>
  );
}
```

El patrón `asChild` de shadcn en `Button` permite que el enlace de React Router sea el elemento real del DOM, conservando la semántica HTML correcta (`<a>` en lugar de `<button>`). El `line-clamp-2` evita que nombres largos rompan el layout de la grilla.

---

## 4.5 Componente CategoryFilter

**Archivo:** `src/presentation/components/CategoryFilter.tsx`

`CategoryFilter` lee las categorías del store y permite al usuario filtrar con un clic. Al desacoplarlo de `CatalogPage`, puede reutilizarse en otras vistas como una barra lateral o un modal.

```typescript
// src/presentation/components/CategoryFilter.tsx

import { useCatalogStore } from '../../domain/store/catalog.store';
import { Button } from '@/components/ui/button';
import { cn } from '@/lib/utils';

export function CategoryFilter() {
  const categories = useCatalogStore((s) => s.categories);
  const filters = useCatalogStore((s) => s.filters);
  const setFilters = useCatalogStore((s) => s.setFilters);

  const handleSelect = (categoryId: number | null) => {
    setFilters({ categoryId });
  };

  return (
    <div className="flex gap-2 overflow-x-auto pb-2 scrollbar-thin">
      <Button
        variant={filters.categoryId === null ? 'default' : 'outline'}
        size="sm"
        className={cn('shrink-0', filters.categoryId === null && 'pointer-events-none')}
        onClick={() => handleSelect(null)}
      >
        Todos
      </Button>

      {categories.map((cat) => (
        <Button
          key={cat.id}
          variant={filters.categoryId === cat.id ? 'default' : 'outline'}
          size="sm"
          className={cn(
            'shrink-0',
            filters.categoryId === cat.id && 'pointer-events-none',
          )}
          onClick={() => handleSelect(cat.id)}
        >
          {cat.name}
          {cat.product_count !== undefined && (
            <span className="ml-1 text-xs opacity-70">({cat.product_count})</span>
          )}
        </Button>
      ))}
    </div>
  );
}
```

La fila usa `overflow-x-auto` para permitir scroll horizontal en pantallas pequeñas sin romper el layout. El botón activo recibe `pointer-events-none` para evitar una llamada redundante a la API cuando ya está seleccionado.

---

## 4.6 Componente ProductCardSkeleton

**Archivo:** `src/presentation/components/ProductCardSkeleton.tsx`

El skeleton mantiene las mismas dimensiones que `ProductCard` para evitar el salto visual (layout shift) cuando los datos reales cargan. Usar shadcn `Skeleton` garantiza consistencia con el tema de la app.

```typescript
// src/presentation/components/ProductCardSkeleton.tsx

import { Card, CardContent, CardFooter, CardHeader } from '@/components/ui/card';
import { Skeleton } from '@/components/ui/skeleton';

export function ProductCardSkeleton() {
  return (
    <Card className="flex flex-col overflow-hidden">
      <CardHeader className="p-0">
        <Skeleton className="h-48 w-full rounded-none" />
      </CardHeader>

      <CardContent className="flex flex-1 flex-col gap-2 p-4">
        <div className="flex items-start justify-between gap-2">
          <Skeleton className="h-4 w-3/4" />
          <Skeleton className="h-5 w-16 rounded-full" />
        </div>
        <Skeleton className="h-6 w-24" />
        <Skeleton className="h-5 w-20 rounded-full" />
      </CardContent>

      <CardFooter className="p-4 pt-0">
        <Skeleton className="h-8 w-full" />
      </CardFooter>
    </Card>
  );
}
```

Renderizar 8 instancias de este componente durante la carga da al usuario retroalimentación visual inmediata y mantiene la estructura de la grilla antes de que lleguen los datos reales.

---

## 4.7 CatalogPage

**Archivo:** `src/presentation/pages/catalog/CatalogPage.tsx`

`CatalogPage` orquesta el ciclo de vida completo del catálogo: carga inicial, renderizado condicional de estados (loading, vacío, datos) y paginación. Es la única página que conoce el store y los componentes de catálogo.

```typescript
// src/presentation/pages/catalog/CatalogPage.tsx

import { useEffect } from 'react';
import { AppShell } from '../../components/AppShell';
import { CategoryFilter } from '../../components/CategoryFilter';
import { ProductCard } from '../../components/ProductCard';
import { ProductCardSkeleton } from '../../components/ProductCardSkeleton';
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

  // Cargar categorías una sola vez al montar
  useEffect(() => {
    if (categories.length === 0) {
      fetchCategories();
    }
  }, []);

  // Recargar productos cuando cambia la página
  useEffect(() => {
    fetchProducts();
  }, [currentPage]);

  const handlePrev = () => {
    if (currentPage > 1) {
      setPage(currentPage - 1);
    }
  };

  const handleNext = () => {
    if (currentPage < totalPages) {
      setPage(currentPage + 1);
    }
  };

  return (
    <AppShell>
      <div className="container mx-auto px-4 py-6">
        <h1 className="mb-4 text-2xl font-bold">Catálogo</h1>

        {/* Filtro de categorías */}
        <div className="mb-6">
          <CategoryFilter />
        </div>

        {/* Grilla de productos */}
        {isLoading ? (
          <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4">
            {Array.from({ length: 8 }).map((_, i) => (
              <ProductCardSkeleton key={i} />
            ))}
          </div>
        ) : products.length === 0 ? (
          <div className="flex min-h-64 items-center justify-center">
            <p className="text-muted-foreground">No se encontraron productos</p>
          </div>
        ) : (
          <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4">
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
    </AppShell>
  );
}
```

El primer `useEffect` llama a `fetchCategories` solo si la lista está vacía, evitando llamadas repetidas al navegar de regreso a esta página. El segundo `useEffect` depende de `currentPage` para que `setPage` desencadene automáticamente la carga de la nueva página.

---

## 4.8 Registro de la ruta en el router

Añade la ruta del catálogo en tu archivo de router existente. Si usas `src/presentation/router/AppRouter.tsx`, agrega:

```typescript
// src/presentation/router/AppRouter.tsx  (fragmento — añadir dentro de <Routes>)

import { CatalogPage } from '../pages/catalog/CatalogPage';

// Dentro de <Routes>:
<Route path="/catalog" element={<CatalogPage />} />
<Route path="/products/:id" element={<div>Detalle (Módulo 5)</div>} />
```

El placeholder en `/products/:id` será reemplazado en el módulo 5. Si el router ya usa un layout con `<Outlet>`, adapta la estructura según corresponda.

---

## 4.9 Checkpoint y tabla resumen

Antes de continuar con el módulo 5, verifica:

- [ ] `http://localhost:5173/catalog` muestra productos del backend.
- [ ] Hacer clic en una categoría filtra la grilla (observa el request en DevTools).
- [ ] "Todos" limpia el filtro de categoría.
- [ ] Los botones Anterior/Siguiente cambian de página correctamente y quedan deshabilitados en los extremos.
- [ ] Durante la carga se muestran 8 tarjetas skeleton.
- [ ] Si el backend devuelve 0 resultados, aparece "No se encontraron productos".
- [ ] `npm run build` compila sin errores.

---

### Tabla resumen del módulo 4

| Archivo | Capa | Responsabilidad |
|---|---|---|
| `src/domain/model/catalog.types.ts` | Domain / Model | Interfaces TypeScript de Category, Product, filtros y paginación |
| `src/data/api/catalog.api.ts` | Data / API | Funciones HTTP para categorías, listado y detalle de productos |
| `src/domain/store/catalog.store.ts` | Domain / Store | Estado global del catálogo: productos, categorías, filtros, página |
| `src/presentation/components/ProductCard.tsx` | Presentation | Tarjeta visual de un producto con imagen, precio, stock y enlace |
| `src/presentation/components/ProductCardSkeleton.tsx` | Presentation | Placeholder animado de carga con las mismas dimensiones que ProductCard |
| `src/presentation/components/CategoryFilter.tsx` | Presentation | Fila de botones para filtrar por categoría |
| `src/presentation/pages/catalog/CatalogPage.tsx` | Presentation | Página principal del catálogo: orquesta carga, estados y paginación |
