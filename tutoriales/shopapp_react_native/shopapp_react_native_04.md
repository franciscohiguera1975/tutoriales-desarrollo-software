# ShopApp React Native — Módulo 4

## Catálogo — Productos, Categorías, Búsqueda y Filtros

---

> **Objetivo**
> Construir la pantalla principal del catálogo de productos con lista paginada, filtro por
> categoría, búsqueda en tiempo real, pull-to-refresh e infinite scroll. Todo el estado se
> maneja en un CatalogStore de Zustand que llama a las funciones de `catalog.api.ts`
> creadas en el módulo 2.
>
> **Checkpoint final**
> - Ver la lista de productos cargada desde el backend.
> - Filtrar por categoría usando las chips horizontales.
> - Buscar productos por nombre y ver los resultados actualizarse.
> - Hacer scroll hasta el final y ver que cargan más productos (infinite scroll).
> - Pull-to-refresh recarga la lista desde el principio.

---

## 4.1 Tipos TypeScript

**Archivo:** `src/domain/model/catalog.types.ts`

Refleja exactamente lo que devuelve la API Django para productos y categorías.

```typescript
// src/domain/model/catalog.types.ts

/**
 * Categoría tal como llega del endpoint GET /api/categories/
 */
export interface Category {
  id:             number;
  name:           string;
  slug:           string;
  description:    string;
  total_products: number;
}

/**
 * Referencia a categoría embebida dentro de un producto.
 */
export interface CategoryRef {
  id:   number;
  name: string;
}

/**
 * Producto en el listado paginado (GET /api/products/).
 * No incluye description ni price_with_tax (esos vienen en el detalle).
 */
export interface Product {
  id:        number;
  name:      string;
  price:     string;   // Django devuelve DecimalField como string
  stock:     number;
  is_active: boolean;
  image_url: string | null;
  category:  CategoryRef;
}

/**
 * Respuesta paginada de Django REST Framework.
 * GET /api/products/?page=N&search=X&category=Y
 */
export interface PaginatedProducts {
  count:    number;
  next:     string | null;
  previous: string | null;
  results:  Product[];
}

/**
 * Filtros que se pueden aplicar al listado de productos.
 * Se pasan como query params al endpoint.
 */
export interface ProductFilters {
  search?:   string;
  category?: number;
  page?:     number;
}
```

> **Sobre `price: string`:** Django serializa `DecimalField` como cadena para preservar la
> precisión decimal. Al mostrar el precio usa `parseFloat(product.price).toFixed(2)` o
> formatea con `Intl.NumberFormat`.

---

## 4.2 CatalogStore con Zustand

**Archivo:** `src/domain/store/catalog.store.ts`

Gestiona categorías, productos, paginación, filtros activos y estados de carga.
Las funciones `getCategoriesApi` y `getProductsApi` provienen de `catalog.api.ts` (módulo 2).

```typescript
// src/domain/store/catalog.store.ts

import { create } from 'zustand';
import { getCategoriesApi, getProductsApi } from '../../data/api/catalog.api';
import type { Category, Product, PaginatedProducts, ProductFilters } from '../model/catalog.types';

// ─── Forma del store ──────────────────────────────────────────────────────────

interface CatalogStore {
  // Estado
  categories:     Category[];
  products:       Product[];
  totalProducts:  number;
  nextPageUrl:    string | null;
  currentFilters: ProductFilters;
  isLoadingCats:  boolean;
  isLoadingProds: boolean;
  isLoadingMore:  boolean;
  error:          string | null;

  // Acciones
  loadCategories: ()                           => Promise<void>;
  loadProducts:   (filters?: ProductFilters)   => Promise<void>;
  loadMore:       ()                           => Promise<void>;
  setFilter:      (filters: ProductFilters)    => void;
  clearFilters:   ()                           => void;
}

// ─── Store ────────────────────────────────────────────────────────────────────

export const useCatalogStore = create<CatalogStore>((set, get) => ({
  // ── Estado inicial ──────────────────────────────────────────────────────────
  categories:     [],
  products:       [],
  totalProducts:  0,
  nextPageUrl:    null,
  currentFilters: {},
  isLoadingCats:  false,
  isLoadingProds: false,
  isLoadingMore:  false,
  error:          null,

  // ── Acciones ────────────────────────────────────────────────────────────────

  /**
   * Carga todas las categorías del backend.
   * Se llama una sola vez al montar CatalogScreen.
   */
  loadCategories: async () => {
    set({ isLoadingCats: true, error: null });
    try {
      const data = await getCategoriesApi();
      set({ categories: data, isLoadingCats: false });
    } catch (err: any) {
      set({
        error: err?.message || 'Error cargando categorías.',
        isLoadingCats: false,
      });
    }
  },

  /**
   * Carga la primera página de productos con los filtros dados.
   * Resetea la lista completa (no es "cargar más").
   */
  loadProducts: async (filters = {}) => {
    set({ isLoadingProds: true, error: null, currentFilters: filters });
    try {
      const data: PaginatedProducts = await getProductsApi(filters);
      set({
        products:      data.results,
        totalProducts: data.count,
        nextPageUrl:   data.next,
        isLoadingProds: false,
      });
    } catch (err: any) {
      set({
        error: err?.message || 'Error cargando productos.',
        isLoadingProds: false,
      });
    }
  },

  /**
   * Carga la siguiente página y agrega los productos al final de la lista.
   * Solo actúa si hay una URL de página siguiente disponible.
   */
  loadMore: async () => {
    const { nextPageUrl, isLoadingMore, isLoadingProds, currentFilters } = get();
    if (!nextPageUrl || isLoadingMore || isLoadingProds) return;

    set({ isLoadingMore: true });
    try {
      // Extraemos el número de página de la URL para pasarlo como filtro
      const url = new URL(nextPageUrl);
      const page = parseInt(url.searchParams.get('page') || '2', 10);
      const data: PaginatedProducts = await getProductsApi({ ...currentFilters, page });
      set((state) => ({
        products:     [...state.products, ...data.results],
        nextPageUrl:  data.next,
        isLoadingMore: false,
      }));
    } catch (err: any) {
      set({ error: err?.message || 'Error cargando más productos.', isLoadingMore: false });
    }
  },

  /**
   * Actualiza los filtros activos y recarga los productos desde la página 1.
   */
  setFilter: (filters) => {
    const merged = { ...get().currentFilters, ...filters };
    get().loadProducts(merged);
  },

  /**
   * Elimina todos los filtros activos y recarga la lista limpia.
   */
  clearFilters: () => {
    get().loadProducts({});
  },
}));
```

> **Por qué `new URL(nextPageUrl)`?**
> Django REST devuelve la URL completa de la siguiente página, por ejemplo
> `http://10.0.2.2:8000/api/products/?page=2&search=camisa`.
> Parseamos la URL para extraer el número de página y re-usar el mismo
> `getProductsApi` en lugar de hacer una petición por URL directa.

---

## 4.3 ProductCard

**Archivo:** `src/presentation/components/ProductCard.tsx`

Tarjeta compacta que muestra imagen, nombre, categoría y precio. Es el elemento de la FlatList.

```typescript
// src/presentation/components/ProductCard.tsx

import React, { memo } from 'react';
import {
  View,
  Text,
  Image,
  TouchableOpacity,
  StyleSheet,
  Dimensions,
} from 'react-native';

import { Colors } from '../../theme/colors';
import type { Product } from '../../domain/model/catalog.types';

const CARD_WIDTH = (Dimensions.get('window').width - 48) / 2; // 2 columnas con margen

interface ProductCardProps {
  product:   Product;
  onPress:   (product: Product) => void;
}

function ProductCard({ product, onPress }: ProductCardProps) {
  const price = parseFloat(product.price).toFixed(2);

  return (
    <TouchableOpacity
      style={styles.card}
      onPress={() => onPress(product)}
      activeOpacity={0.85}
    >
      {/* Imagen del producto */}
      {product.image_url ? (
        <Image
          source={{ uri: product.image_url }}
          style={styles.image}
          resizeMode="cover"
        />
      ) : (
        <View style={[styles.image, styles.imagePlaceholder]}>
          <Text style={styles.placeholderText}>Sin imagen</Text>
        </View>
      )}

      {/* Información */}
      <View style={styles.info}>
        {/* Badge de categoría */}
        <View style={styles.categoryBadge}>
          <Text style={styles.categoryText} numberOfLines={1}>
            {product.category.name}
          </Text>
        </View>

        {/* Nombre */}
        <Text style={styles.name} numberOfLines={2}>
          {product.name}
        </Text>

        {/* Precio y disponibilidad */}
        <View style={styles.footer}>
          <Text style={styles.price}>${price}</Text>
          {product.stock === 0 && (
            <Text style={styles.outOfStock}>Agotado</Text>
          )}
        </View>
      </View>
    </TouchableOpacity>
  );
}

// memo evita re-renders cuando el producto no cambia
export default memo(ProductCard);

const styles = StyleSheet.create({
  card: {
    width:           CARD_WIDTH,
    backgroundColor: Colors.surface,
    borderRadius:    12,
    overflow:        'hidden',
    borderWidth:     1,
    borderColor:     Colors.border,
    margin:          6,
  },
  image: {
    width:  '100%',
    height: CARD_WIDTH * 0.75,
  },
  imagePlaceholder: {
    backgroundColor: Colors.surface2,
    justifyContent:  'center',
    alignItems:      'center',
  },
  placeholderText: {
    color:    Colors.textSecondary,
    fontSize: 12,
  },
  info: {
    padding: 10,
    gap:     4,
  },
  categoryBadge: {
    backgroundColor: Colors.accent + '30', // 30 = ~18% opacidad
    borderRadius:    4,
    paddingHorizontal: 6,
    paddingVertical:   2,
    alignSelf:       'flex-start',
  },
  categoryText: {
    color:     Colors.accent,
    fontSize:  10,
    fontWeight: '600',
  },
  name: {
    color:      Colors.textPrimary,
    fontSize:   13,
    fontWeight: '600',
    lineHeight: 18,
  },
  footer: {
    flexDirection:  'row',
    alignItems:     'center',
    justifyContent: 'space-between',
    marginTop:      4,
  },
  price: {
    color:      Colors.accent,
    fontSize:   15,
    fontWeight: '800',
  },
  outOfStock: {
    color:     Colors.error,
    fontSize:  11,
    fontWeight: '600',
  },
});
```

---

## 4.4 CategoryChip

**Archivo:** `src/presentation/components/CategoryChip.tsx`

Chip/píldora para seleccionar o deseleccionar una categoría como filtro activo.

```typescript
// src/presentation/components/CategoryChip.tsx

import React, { memo } from 'react';
import { TouchableOpacity, Text, StyleSheet } from 'react-native';
import { Colors } from '../../theme/colors';
import type { Category } from '../../domain/model/catalog.types';

interface CategoryChipProps {
  category:   Category;
  isSelected: boolean;
  onPress:    (category: Category) => void;
}

function CategoryChip({ category, isSelected, onPress }: CategoryChipProps) {
  return (
    <TouchableOpacity
      style={[styles.chip, isSelected && styles.chipSelected]}
      onPress={() => onPress(category)}
      activeOpacity={0.7}
    >
      <Text style={[styles.label, isSelected && styles.labelSelected]}>
        {category.name}
      </Text>
      {/* Contador de productos de la categoría */}
      <Text style={[styles.count, isSelected && styles.countSelected]}>
        {category.total_products}
      </Text>
    </TouchableOpacity>
  );
}

export default memo(CategoryChip);

const styles = StyleSheet.create({
  chip: {
    flexDirection:    'row',
    alignItems:       'center',
    backgroundColor:  Colors.surface,
    borderWidth:      1,
    borderColor:      Colors.border,
    borderRadius:     20,
    paddingVertical:  7,
    paddingHorizontal: 14,
    marginRight:      8,
    gap:              6,
  },
  chipSelected: {
    backgroundColor: Colors.accent,
    borderColor:     Colors.accent,
  },
  label: {
    color:      Colors.textSecondary,
    fontSize:   13,
    fontWeight: '600',
  },
  labelSelected: {
    color: Colors.textPrimary,
  },
  count: {
    color:           Colors.textSecondary,
    fontSize:        11,
    backgroundColor: Colors.surface2,
    borderRadius:    10,
    paddingHorizontal: 6,
    paddingVertical:   1,
    overflow:        'hidden',
  },
  countSelected: {
    color:           Colors.accent,
    backgroundColor: Colors.textPrimary + '20',
  },
});
```

---

## 4.5 CatalogScreen

**Archivo:** `src/presentation/screens/catalog/CatalogScreen.tsx`

Pantalla principal con barra de búsqueda, chips de categoría y lista de productos en dos
columnas con infinite scroll y pull-to-refresh.

```typescript
// src/presentation/screens/catalog/CatalogScreen.tsx

import React, { useEffect, useState, useCallback, useRef } from 'react';
import {
  View,
  TextInput,
  FlatList,
  ScrollView,
  ActivityIndicator,
  Text,
  StyleSheet,
  RefreshControl,
  TouchableOpacity,
} from 'react-native';
import type { NativeStackScreenProps } from '@react-navigation/native-stack';

import { useCatalogStore } from '../../../domain/store/catalog.store';
import ProductCard          from '../../components/ProductCard';
import CategoryChip         from '../../components/CategoryChip';
import { Colors }           from '../../../theme/colors';
import type { Product, Category } from '../../../domain/model/catalog.types';
import type { CatalogStackParamList } from '../../navigation/MainNavigator';

type Props = NativeStackScreenProps<CatalogStackParamList, 'Catalog'>;

// Tiempo de espera (ms) antes de disparar la búsqueda al escribir
const SEARCH_DEBOUNCE = 400;

export default function CatalogScreen({ navigation }: Props) {
  const {
    categories,
    products,
    isLoadingCats,
    isLoadingProds,
    isLoadingMore,
    error,
    loadCategories,
    loadProducts,
    loadMore,
  } = useCatalogStore();

  const [searchText, setSearchText]         = useState('');
  const [selectedCategory, setSelectedCategory] = useState<number | undefined>(undefined);
  const searchTimer = useRef<ReturnType<typeof setTimeout> | null>(null);

  // ── Carga inicial ────────────────────────────────────────────────────────────
  useEffect(() => {
    loadCategories();
    loadProducts({});
  }, []);  // eslint-disable-line react-hooks/exhaustive-deps

  // ── Búsqueda con debounce ────────────────────────────────────────────────────
  const handleSearchChange = (text: string) => {
    setSearchText(text);
    if (searchTimer.current) clearTimeout(searchTimer.current);
    searchTimer.current = setTimeout(() => {
      loadProducts({ search: text || undefined, category: selectedCategory });
    }, SEARCH_DEBOUNCE);
  };

  // ── Filtro por categoría ─────────────────────────────────────────────────────
  const handleCategoryPress = (category: Category) => {
    const newCat = selectedCategory === category.id ? undefined : category.id;
    setSelectedCategory(newCat);
    loadProducts({ search: searchText || undefined, category: newCat });
  };

  // ── Limpiar filtros ──────────────────────────────────────────────────────────
  const handleClearFilters = () => {
    setSearchText('');
    setSelectedCategory(undefined);
    loadProducts({});
  };

  // ── Pull-to-refresh ──────────────────────────────────────────────────────────
  const handleRefresh = useCallback(() => {
    loadProducts({ search: searchText || undefined, category: selectedCategory });
  }, [searchText, selectedCategory]);  // eslint-disable-line react-hooks/exhaustive-deps

  // ── Navegar al detalle ───────────────────────────────────────────────────────
  const handleProductPress = (product: Product) => {
    navigation.navigate('ProductDetail', { productId: product.id });
  };

  // ── Footer de la FlatList ────────────────────────────────────────────────────
  const renderFooter = () => {
    if (!isLoadingMore) return null;
    return (
      <View style={styles.footerLoader}>
        <ActivityIndicator color={Colors.accent} />
      </View>
    );
  };

  // ── Estado vacío ─────────────────────────────────────────────────────────────
  const renderEmpty = () => {
    if (isLoadingProds) return null;
    return (
      <View style={styles.emptyContainer}>
        <Text style={styles.emptyText}>
          {error ? `Error: ${error}` : 'No se encontraron productos.'}
        </Text>
        {(searchText || selectedCategory) && (
          <TouchableOpacity onPress={handleClearFilters} style={styles.clearButton}>
            <Text style={styles.clearButtonText}>Limpiar filtros</Text>
          </TouchableOpacity>
        )}
      </View>
    );
  };

  const hasActiveFilters = !!searchText || selectedCategory !== undefined;

  return (
    <View style={styles.container}>

      {/* ── Encabezado ─────────────────────────────────────────────────────── */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>Catálogo</Text>
        {hasActiveFilters && (
          <TouchableOpacity onPress={handleClearFilters}>
            <Text style={styles.clearText}>Limpiar</Text>
          </TouchableOpacity>
        )}
      </View>

      {/* ── Barra de búsqueda ──────────────────────────────────────────────── */}
      <View style={styles.searchContainer}>
        <TextInput
          style={styles.searchInput}
          placeholder="Buscar productos..."
          placeholderTextColor={Colors.textSecondary}
          value={searchText}
          onChangeText={handleSearchChange}
          autoCorrect={false}
          returnKeyType="search"
          clearButtonMode="while-editing"
        />
      </View>

      {/* ── Chips de categoría ─────────────────────────────────────────────── */}
      {isLoadingCats ? (
        <ActivityIndicator
          style={styles.catsLoader}
          color={Colors.accent}
          size="small"
        />
      ) : (
        <ScrollView
          horizontal
          showsHorizontalScrollIndicator={false}
          style={styles.chipsScroll}
          contentContainerStyle={styles.chipsContent}
        >
          {categories.map((cat) => (
            <CategoryChip
              key={cat.id}
              category={cat}
              isSelected={selectedCategory === cat.id}
              onPress={handleCategoryPress}
            />
          ))}
        </ScrollView>
      )}

      {/* ── Indicador de carga principal ───────────────────────────────────── */}
      {isLoadingProds && products.length === 0 && (
        <View style={styles.mainLoader}>
          <ActivityIndicator size="large" color={Colors.accent} />
        </View>
      )}

      {/* ── Lista de productos ─────────────────────────────────────────────── */}
      <FlatList
        data={products}
        keyExtractor={(item) => String(item.id)}
        numColumns={2}
        columnWrapperStyle={styles.row}
        contentContainerStyle={styles.listContent}
        renderItem={({ item }) => (
          <ProductCard product={item} onPress={handleProductPress} />
        )}
        onEndReached={loadMore}
        onEndReachedThreshold={0.4}
        ListFooterComponent={renderFooter}
        ListEmptyComponent={renderEmpty}
        refreshControl={
          <RefreshControl
            refreshing={isLoadingProds && products.length > 0}
            onRefresh={handleRefresh}
            tintColor={Colors.accent}
            colors={[Colors.accent]}
          />
        }
        showsVerticalScrollIndicator={false}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex:            1,
    backgroundColor: Colors.background,
  },
  header: {
    flexDirection:  'row',
    alignItems:     'center',
    justifyContent: 'space-between',
    paddingHorizontal: 16,
    paddingTop:     16,
    paddingBottom:  8,
  },
  headerTitle: {
    color:      Colors.textPrimary,
    fontSize:   24,
    fontWeight: '800',
  },
  clearText: {
    color:     Colors.accent,
    fontSize:  14,
    fontWeight: '600',
  },
  searchContainer: {
    paddingHorizontal: 16,
    paddingBottom:     12,
  },
  searchInput: {
    backgroundColor:   Colors.surface,
    borderWidth:       1,
    borderColor:       Colors.border,
    borderRadius:      10,
    paddingHorizontal: 16,
    paddingVertical:   11,
    fontSize:          15,
    color:             Colors.textPrimary,
  },
  chipsScroll: {
    flexGrow: 0,
  },
  chipsContent: {
    paddingHorizontal: 16,
    paddingBottom:     12,
  },
  catsLoader: {
    marginVertical: 12,
  },
  mainLoader: {
    flex:           1,
    justifyContent: 'center',
    alignItems:     'center',
  },
  row: {
    justifyContent: 'center',
  },
  listContent: {
    paddingHorizontal: 10,
    paddingBottom:     24,
    flexGrow:          1,
  },
  footerLoader: {
    paddingVertical: 20,
    alignItems:      'center',
  },
  emptyContainer: {
    flex:           1,
    justifyContent: 'center',
    alignItems:     'center',
    paddingTop:     60,
    gap:            16,
  },
  emptyText: {
    color:     Colors.textSecondary,
    fontSize:  15,
    textAlign: 'center',
  },
  clearButton: {
    backgroundColor: Colors.accent,
    paddingHorizontal: 20,
    paddingVertical:   10,
    borderRadius:      8,
  },
  clearButtonText: {
    color:      Colors.textPrimary,
    fontWeight: '700',
  },
});
```

---

## 4.6 Integrar en MainNavigator

**Archivo:** `src/presentation/navigation/MainNavigator.tsx`

Reemplaza el placeholder del módulo 3 con un Stack navigator anidado dentro de un Bottom Tab
navigator. En este módulo solo hay una pestaña (Catálogo); en el módulo 5 se agregan Detalle
y Carrito.

```typescript
// src/presentation/navigation/MainNavigator.tsx

import React from 'react';
import { TouchableOpacity, Text, StyleSheet, View } from 'react-native';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

import CatalogScreen     from '../screens/catalog/CatalogScreen';
import { useAuthStore }  from '../../domain/store/auth.store';
import { Colors }        from '../../theme/colors';

// ─── Tipos de rutas ───────────────────────────────────────────────────────────

/**
 * Rutas del stack de catálogo. ProductDetail se agrega en el módulo 5.
 */
export type CatalogStackParamList = {
  Catalog:       undefined;
  ProductDetail: { productId: number };
};

/**
 * Pestañas del bottom tab navigator.
 */
export type MainTabParamList = {
  CatalogTab: undefined;
  CartTab:    undefined;   // Se implementa en el módulo 5
};

// ─── Navegadores ──────────────────────────────────────────────────────────────

const CatalogStack = createNativeStackNavigator<CatalogStackParamList>();
const Tab          = createBottomTabNavigator<MainTabParamList>();

/**
 * Stack del catálogo (Catalog → ProductDetail).
 * El header muestra el botón de logout para este módulo.
 */
function CatalogStackNavigator() {
  const { user, logout } = useAuthStore();

  return (
    <CatalogStack.Navigator
      screenOptions={{
        headerStyle:     { backgroundColor: Colors.surface },
        headerTintColor: Colors.textPrimary,
        headerTitleStyle: { fontWeight: '700' },
        contentStyle:    { backgroundColor: Colors.background },
      }}
    >
      <CatalogStack.Screen
        name="Catalog"
        component={CatalogScreen}
        options={{
          headerShown: false,   // CatalogScreen tiene su propio encabezado
        }}
      />
      {/* ProductDetail se agrega en el Módulo 5 */}
    </CatalogStack.Navigator>
  );
}

/**
 * Placeholder para la pestaña del carrito (Módulo 5).
 */
function CartPlaceholder() {
  return (
    <View style={styles.placeholder}>
      <Text style={styles.placeholderText}>Carrito — Módulo 5</Text>
    </View>
  );
}

// ─── MainNavigator ────────────────────────────────────────────────────────────

export default function MainNavigator() {
  const { logout } = useAuthStore();

  return (
    <Tab.Navigator
      screenOptions={{
        headerShown:     false,
        tabBarStyle:     styles.tabBar,
        tabBarActiveTintColor:   Colors.accent,
        tabBarInactiveTintColor: Colors.textSecondary,
        tabBarLabelStyle: styles.tabBarLabel,
      }}
    >
      <Tab.Screen
        name="CatalogTab"
        component={CatalogStackNavigator}
        options={{
          tabBarLabel: 'Catálogo',
          tabBarIcon: ({ color }) => (
            <Text style={{ color, fontSize: 20 }}>🛍</Text>
          ),
        }}
      />
      <Tab.Screen
        name="CartTab"
        component={CartPlaceholder}
        options={{
          tabBarLabel: 'Carrito',
          tabBarIcon: ({ color }) => (
            <Text style={{ color, fontSize: 20 }}>🛒</Text>
          ),
        }}
      />
    </Tab.Navigator>
  );
}

const styles = StyleSheet.create({
  tabBar: {
    backgroundColor: Colors.surface,
    borderTopColor:  Colors.border,
    borderTopWidth:  1,
    height:          60,
    paddingBottom:   8,
  },
  tabBarLabel: {
    fontSize:   11,
    fontWeight: '600',
  },
  placeholder: {
    flex:            1,
    backgroundColor: Colors.background,
    justifyContent:  'center',
    alignItems:      'center',
  },
  placeholderText: {
    color:    Colors.textSecondary,
    fontSize: 16,
  },
});
```

> **Estructura de navegación completa en este módulo:**
>
> ```
> NavigationContainer
>   AppNavigator
>     ├── AuthNavigator (si no autenticado)
>     │     ├── LoginScreen
>     │     └── RegisterScreen
>     └── MainNavigator (si autenticado)
>           └── Tab: CatalogTab
>                 └── CatalogStack
>                       └── CatalogScreen
> ```

---

## Resumen de archivos del Módulo 4

| Archivo | Descripción |
|---|---|
| `src/domain/model/catalog.types.ts` | Interfaces TypeScript del catálogo |
| `src/domain/store/catalog.store.ts` | Store Zustand: estado de catálogo y acciones |
| `src/presentation/components/ProductCard.tsx` | Tarjeta de producto para FlatList |
| `src/presentation/components/CategoryChip.tsx` | Chip de filtro de categoría |
| `src/presentation/screens/catalog/CatalogScreen.tsx` | Pantalla principal del catálogo |
| `src/presentation/navigation/MainNavigator.tsx` | Bottom tabs con stack de catálogo |

---

## Checkpoint final — Módulo 4

**Prueba 1: Carga de productos**
1. Inicia sesión y navega a la pestaña Catálogo.
2. Deben aparecer las tarjetas de productos con imagen, nombre y precio.
3. El indicador de carga (`ActivityIndicator`) debe aparecer mientras llegan los datos.

**Prueba 2: Filtro por categoría**
1. Espera a que carguen las chips de categoría.
2. Toca una categoría: las chips y la lista deben actualizarse.
3. Toca la misma categoría de nuevo: el filtro se deselecciona y aparecen todos los productos.

**Prueba 3: Búsqueda**
1. Escribe en la barra de búsqueda (p. ej. "camisa").
2. Tras el debounce de 400 ms la lista se actualiza con los resultados.
3. Borra el texto: la lista vuelve al estado sin filtro.

**Prueba 4: Infinite scroll**
1. Desplázate hasta el final de la lista.
2. Debe aparecer un `ActivityIndicator` y cargar más productos.
3. Si no hay más páginas, no debe ocurrir nada.

**Prueba 5: Pull-to-refresh**
1. Arrastra la lista hacia abajo desde el principio.
2. La lista se recarga con los filtros activos.

**Prueba 6: Estado vacío**
1. Busca un término que no tenga resultados (p. ej. "xyzzy").
2. Debe mostrarse el mensaje "No se encontraron productos." con el botón "Limpiar filtros".
