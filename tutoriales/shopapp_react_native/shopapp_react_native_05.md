# ShopApp React Native — Módulo 5

## Detalle de Producto y Carrito

---

> **Objetivo**
> Implementar la pantalla de detalle de producto con toda la información completa que devuelve
> el backend (descripción, precio con impuesto, estado de stock), un store de carrito en Zustand
> que persiste en memoria durante la sesión, la pantalla del carrito con controles de cantidad
> y total, y el ícono con badge en el header que muestra cuántos ítems hay en el carrito.
>
> **Checkpoint final**
> - Tocar un producto en el catálogo y ver su detalle completo con imagen grande.
> - Seleccionar una cantidad y agregarlo al carrito.
> - El badge del header actualiza instantáneamente.
> - Navegar al carrito, ajustar cantidades, eliminar ítems y ver el total recalculado.
> - El botón "Proceder al pago" muestra el total final (se conectará al checkout en un módulo posterior).

---

## 5.1 Tipos del carrito

**Archivo:** `src/domain/model/cart.types.ts`

Define la estructura de cada ítem del carrito y el estado global del store.

```typescript
// src/domain/model/cart.types.ts

import type { Product } from './catalog.types';

/**
 * Un ítem en el carrito: producto + cantidad seleccionada.
 * Guardamos el snapshot del producto para no perder los datos si el catálogo cambia.
 */
export interface CartItem {
  product:  Product;
  quantity: number;
}

/**
 * Estado del CartStore.
 * - items: lista de ítems en el carrito.
 * - itemCount: total de unidades (suma de quantities).
 * - total: precio total en pesos/dólares (string formateado).
 */
export interface CartState {
  items:     CartItem[];
  itemCount: number;
  total:     string;
}
```

---

## 5.2 CartStore con Zustand

**Archivo:** `src/domain/store/cart.store.ts`

Store de carrito en memoria. Los valores calculados (`itemCount`, `total`) se recomputan en cada
acción para que los componentes siempre lean datos consistentes.

```typescript
// src/domain/store/cart.store.ts

import { create } from 'zustand';
import type { CartItem } from '../model/cart.types';
import type { Product }  from '../model/catalog.types';

// ─── Helpers de cálculo ───────────────────────────────────────────────────────

/** Suma el total de unidades de todos los ítems. */
function computeItemCount(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.quantity, 0);
}

/**
 * Calcula el precio total como string con 2 decimales.
 * Usa parseFloat porque Django devuelve DecimalField como string.
 */
function computeTotal(items: CartItem[]): string {
  const raw = items.reduce(
    (sum, item) => sum + parseFloat(item.product.price) * item.quantity,
    0,
  );
  return raw.toFixed(2);
}

// ─── Forma del store ──────────────────────────────────────────────────────────

interface CartStore {
  // Estado
  items:     CartItem[];
  itemCount: number;
  total:     string;

  // Acciones
  addItem:        (product: Product, quantity?: number) => void;
  removeItem:     (productId: number) => void;
  updateQuantity: (productId: number, quantity: number) => void;
  clearCart:      () => void;
}

// ─── Store ────────────────────────────────────────────────────────────────────

export const useCartStore = create<CartStore>((set, get) => ({
  // ── Estado inicial ──────────────────────────────────────────────────────────
  items:     [],
  itemCount: 0,
  total:     '0.00',

  // ── Acciones ────────────────────────────────────────────────────────────────

  /**
   * Agrega un producto al carrito.
   * - Si ya existe, incrementa su cantidad.
   * - Si es nuevo, lo agrega al final de la lista.
   * - No permite superar el stock disponible.
   */
  addItem: (product, quantity = 1) => {
    const { items } = get();
    const existing = items.find((item) => item.product.id === product.id);

    let updated: CartItem[];
    if (existing) {
      const newQty = Math.min(existing.quantity + quantity, product.stock);
      updated = items.map((item) =>
        item.product.id === product.id ? { ...item, quantity: newQty } : item,
      );
    } else {
      updated = [...items, { product, quantity: Math.min(quantity, product.stock) }];
    }

    set({
      items:     updated,
      itemCount: computeItemCount(updated),
      total:     computeTotal(updated),
    });
  },

  /**
   * Elimina completamente un producto del carrito.
   */
  removeItem: (productId) => {
    const updated = get().items.filter((item) => item.product.id !== productId);
    set({
      items:     updated,
      itemCount: computeItemCount(updated),
      total:     computeTotal(updated),
    });
  },

  /**
   * Cambia la cantidad de un ítem específico.
   * Si la cantidad llega a 0 o menos, elimina el ítem.
   * No permite superar el stock disponible.
   */
  updateQuantity: (productId, quantity) => {
    if (quantity <= 0) {
      get().removeItem(productId);
      return;
    }
    const updated = get().items.map((item) => {
      if (item.product.id !== productId) return item;
      return { ...item, quantity: Math.min(quantity, item.product.stock) };
    });
    set({
      items:     updated,
      itemCount: computeItemCount(updated),
      total:     computeTotal(updated),
    });
  },

  /**
   * Vacía el carrito por completo.
   * Se llama tras completar un pedido o al cerrar sesión.
   */
  clearCart: () => set({ items: [], itemCount: 0, total: '0.00' }),
}));
```

> **Persistencia del carrito:** En este módulo el carrito vive solo en memoria (se limpia al
> cerrar la app). Si quisieras persistirlo, podrías agregar `EncryptedStorage.setItem` al final
> de cada acción, igual que en el `auth.store.ts` del módulo 3.

---

## 5.3 ProductDetailScreen

**Archivo:** `src/presentation/screens/catalog/ProductDetailScreen.tsx`

Pantalla de detalle con imagen a pantalla completa, información completa del producto y
selector de cantidad para agregar al carrito.

```typescript
// src/presentation/screens/catalog/ProductDetailScreen.tsx

import React, { useEffect, useState } from 'react';
import {
  View,
  Text,
  Image,
  ScrollView,
  TouchableOpacity,
  ActivityIndicator,
  StyleSheet,
  Dimensions,
  Alert,
} from 'react-native';
import type { NativeStackScreenProps } from '@react-navigation/native-stack';

import { getProductByIdApi } from '../../../infrastructure/api/catalog.api';
import { useCartStore }      from '../../../domain/store/cart.store';
import { Colors }            from '../../../core/theme/colors';
import type { CatalogStackParamList } from '../../../navigation/MainNavigator';

// Tipo de detalle de producto (incluye campos adicionales del endpoint /api/products/{id}/)
interface ProductDetail {
  id:             number;
  name:           string;
  price:          string;
  price_with_tax: string;
  stock:          number;
  in_stock:       boolean;
  is_active:      boolean;
  image_url:      string | null;
  description:    string;
  category: {
    id:   number;
    name: string;
    slug: string;
  };
}

type Props = NativeStackScreenProps<CatalogStackParamList, 'ProductDetail'>;

const { width: SCREEN_WIDTH } = Dimensions.get('window');
const IMAGE_HEIGHT = SCREEN_WIDTH * 0.75;

export default function ProductDetailScreen({ route, navigation }: Props) {
  const { productId } = route.params;

  const [product, setProduct]   = useState<ProductDetail | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError]         = useState<string | null>(null);
  const [quantity, setQuantity]   = useState(1);

  const { addItem, items } = useCartStore();

  // Carga el detalle del producto al montar la pantalla
  useEffect(() => {
    let cancelled = false;
    (async () => {
      try {
        const data = await getProductByIdApi(productId);
        if (!cancelled) setProduct(data as ProductDetail);
      } catch (err: any) {
        if (!cancelled) setError(err?.message || 'Error cargando el producto.');
      } finally {
        if (!cancelled) setIsLoading(false);
      }
    })();
    return () => { cancelled = true; };
  }, [productId]);

  // Cantidad actual de este producto en el carrito
  const cartItem = items.find((i) => i.product.id === productId);
  const inCartQty = cartItem?.quantity ?? 0;

  const handleAddToCart = () => {
    if (!product) return;
    const totalAfterAdd = inCartQty + quantity;
    if (totalAfterAdd > product.stock) {
      Alert.alert(
        'Stock insuficiente',
        `Solo hay ${product.stock} unidades disponibles y ya tienes ${inCartQty} en el carrito.`,
      );
      return;
    }
    addItem(product as any, quantity);
    Alert.alert(
      'Agregado al carrito',
      `${quantity} × ${product.name}`,
      [
        { text: 'Seguir comprando' },
        { text: 'Ver carrito', onPress: () => navigation.navigate('Cart' as any) },
      ],
    );
  };

  const decreaseQty = () => setQuantity((q) => Math.max(1, q - 1));
  const increaseQty = () => {
    if (!product) return;
    setQuantity((q) => Math.min(product.stock - inCartQty, q + 1));
  };

  // ── Estados de UI ────────────────────────────────────────────────────────────

  if (isLoading) {
    return (
      <View style={styles.centered}>
        <ActivityIndicator size="large" color={Colors.accent} />
      </View>
    );
  }

  if (error || !product) {
    return (
      <View style={styles.centered}>
        <Text style={styles.errorText}>{error || 'Producto no encontrado.'}</Text>
        <TouchableOpacity onPress={() => navigation.goBack()} style={styles.backButton}>
          <Text style={styles.backButtonText}>Volver</Text>
        </TouchableOpacity>
      </View>
    );
  }

  const canAddMore = product.stock - inCartQty > 0;

  return (
    <ScrollView style={styles.container} showsVerticalScrollIndicator={false}>

      {/* ── Imagen ─────────────────────────────────────────────────────────── */}
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

      {/* ── Contenido ──────────────────────────────────────────────────────── */}
      <View style={styles.content}>

        {/* Categoría */}
        <View style={styles.categoryBadge}>
          <Text style={styles.categoryText}>{product.category.name}</Text>
        </View>

        {/* Nombre */}
        <Text style={styles.name}>{product.name}</Text>

        {/* Precios */}
        <View style={styles.priceRow}>
          <Text style={styles.price}>${parseFloat(product.price).toFixed(2)}</Text>
          <Text style={styles.priceWithTax}>
            Con IVA: ${parseFloat(product.price_with_tax).toFixed(2)}
          </Text>
        </View>

        {/* Estado de stock */}
        <View style={styles.stockRow}>
          <View style={[
            styles.stockBadge,
            product.in_stock ? styles.inStock : styles.outOfStock,
          ]}>
            <Text style={styles.stockText}>
              {product.in_stock ? `En stock (${product.stock})` : 'Agotado'}
            </Text>
          </View>
          {inCartQty > 0 && (
            <Text style={styles.inCartText}>{inCartQty} en tu carrito</Text>
          )}
        </View>

        {/* Descripción */}
        {product.description ? (
          <View style={styles.descriptionBlock}>
            <Text style={styles.sectionTitle}>Descripción</Text>
            <Text style={styles.description}>{product.description}</Text>
          </View>
        ) : null}

        {/* Selector de cantidad + botón */}
        {canAddMore ? (
          <>
            <Text style={styles.sectionTitle}>Cantidad</Text>
            <View style={styles.quantityRow}>
              <TouchableOpacity
                style={styles.qtyButton}
                onPress={decreaseQty}
                disabled={quantity <= 1}
              >
                <Text style={styles.qtyButtonText}>−</Text>
              </TouchableOpacity>

              <Text style={styles.qtyValue}>{quantity}</Text>

              <TouchableOpacity
                style={styles.qtyButton}
                onPress={increaseQty}
                disabled={quantity >= product.stock - inCartQty}
              >
                <Text style={styles.qtyButtonText}>+</Text>
              </TouchableOpacity>
            </View>

            <TouchableOpacity
              style={styles.addButton}
              onPress={handleAddToCart}
              activeOpacity={0.85}
            >
              <Text style={styles.addButtonText}>
                Agregar al carrito — ${(parseFloat(product.price) * quantity).toFixed(2)}
              </Text>
            </TouchableOpacity>
          </>
        ) : (
          <View style={styles.unavailableBox}>
            <Text style={styles.unavailableText}>
              {product.in_stock
                ? 'Ya tienes el máximo disponible en tu carrito.'
                : 'Este producto no está disponible.'}
            </Text>
          </View>
        )}

      </View>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex:            1,
    backgroundColor: Colors.background,
  },
  centered: {
    flex:            1,
    backgroundColor: Colors.background,
    justifyContent:  'center',
    alignItems:      'center',
    gap:             16,
    padding:         24,
  },
  image: {
    width:  SCREEN_WIDTH,
    height: IMAGE_HEIGHT,
  },
  imagePlaceholder: {
    backgroundColor: Colors.surface2,
    justifyContent:  'center',
    alignItems:      'center',
  },
  placeholderText: {
    color:    Colors.textSecondary,
    fontSize: 16,
  },
  content: {
    padding: 20,
    gap:     12,
  },
  categoryBadge: {
    backgroundColor:   Colors.accent + '25',
    borderRadius:      6,
    paddingHorizontal: 10,
    paddingVertical:   4,
    alignSelf:         'flex-start',
  },
  categoryText: {
    color:      Colors.accent,
    fontSize:   12,
    fontWeight: '700',
    textTransform: 'uppercase',
    letterSpacing: 0.5,
  },
  name: {
    color:      Colors.textPrimary,
    fontSize:   22,
    fontWeight: '800',
    lineHeight: 30,
  },
  priceRow: {
    flexDirection: 'row',
    alignItems:    'baseline',
    gap:           12,
  },
  price: {
    color:      Colors.accent,
    fontSize:   26,
    fontWeight: '900',
  },
  priceWithTax: {
    color:    Colors.textSecondary,
    fontSize: 13,
  },
  stockRow: {
    flexDirection: 'row',
    alignItems:    'center',
    gap:           12,
  },
  stockBadge: {
    borderRadius:      6,
    paddingHorizontal: 10,
    paddingVertical:   5,
  },
  inStock: {
    backgroundColor: Colors.success + '25',
  },
  outOfStock: {
    backgroundColor: Colors.error + '25',
  },
  stockText: {
    color:      Colors.textPrimary,
    fontSize:   13,
    fontWeight: '600',
  },
  inCartText: {
    color:    Colors.textSecondary,
    fontSize: 13,
  },
  sectionTitle: {
    color:      Colors.textSecondary,
    fontSize:   12,
    fontWeight: '700',
    textTransform: 'uppercase',
    letterSpacing: 0.8,
    marginTop:     4,
  },
  descriptionBlock: {
    gap: 6,
  },
  description: {
    color:      Colors.textSecondary,
    fontSize:   14,
    lineHeight: 22,
  },
  quantityRow: {
    flexDirection:  'row',
    alignItems:     'center',
    gap:            20,
    marginVertical: 4,
  },
  qtyButton: {
    backgroundColor: Colors.surface2,
    width:           40,
    height:          40,
    borderRadius:    8,
    justifyContent:  'center',
    alignItems:      'center',
    borderWidth:     1,
    borderColor:     Colors.border,
  },
  qtyButtonText: {
    color:      Colors.textPrimary,
    fontSize:   22,
    fontWeight: '600',
    lineHeight: 26,
  },
  qtyValue: {
    color:      Colors.textPrimary,
    fontSize:   20,
    fontWeight: '700',
    minWidth:   30,
    textAlign:  'center',
  },
  addButton: {
    backgroundColor: Colors.accent,
    borderRadius:    12,
    paddingVertical: 16,
    alignItems:      'center',
    marginTop:       8,
  },
  addButtonText: {
    color:      Colors.textPrimary,
    fontSize:   16,
    fontWeight: '800',
  },
  unavailableBox: {
    backgroundColor: Colors.surface2,
    borderRadius:    10,
    padding:         16,
    borderWidth:     1,
    borderColor:     Colors.border,
    marginTop:       8,
  },
  unavailableText: {
    color:     Colors.textSecondary,
    fontSize:  14,
    textAlign: 'center',
  },
  errorText: {
    color:     Colors.error,
    fontSize:  15,
    textAlign: 'center',
  },
  backButton: {
    backgroundColor: Colors.accent,
    paddingHorizontal: 24,
    paddingVertical:   12,
    borderRadius:      8,
  },
  backButtonText: {
    color:      Colors.textPrimary,
    fontWeight: '700',
  },
});
```

---

## 5.4 CartIcon badge

**Archivo:** `src/presentation/components/CartIcon.tsx`

Componente de ícono de carrito con badge numérico que se coloca en el header del navigator.
Lee `itemCount` directamente del `useCartStore`.

```typescript
// src/presentation/components/CartIcon.tsx

import React from 'react';
import { TouchableOpacity, Text, View, StyleSheet } from 'react-native';
import { useCartStore } from '../../domain/store/cart.store';
import { Colors } from '../../core/theme/colors';

interface CartIconProps {
  onPress: () => void;
}

export default function CartIcon({ onPress }: CartIconProps) {
  const itemCount = useCartStore((state) => state.itemCount);

  return (
    <TouchableOpacity onPress={onPress} style={styles.container} activeOpacity={0.7}>
      <Text style={styles.icon}>🛒</Text>
      {itemCount > 0 && (
        <View style={styles.badge}>
          <Text style={styles.badgeText}>
            {itemCount > 99 ? '99+' : String(itemCount)}
          </Text>
        </View>
      )}
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  container: {
    marginRight: 4,
    padding:     4,
  },
  icon: {
    fontSize: 24,
  },
  badge: {
    position:         'absolute',
    top:              -2,
    right:            -2,
    backgroundColor:  Colors.error,
    borderRadius:     10,
    minWidth:         18,
    height:           18,
    justifyContent:   'center',
    alignItems:       'center',
    paddingHorizontal: 3,
  },
  badgeText: {
    color:      Colors.textPrimary,
    fontSize:   10,
    fontWeight: '800',
  },
});
```

---

## 5.5 CartScreen

**Archivo:** `src/presentation/screens/cart/CartScreen.tsx`

Lista de ítems del carrito con controles para ajustar cantidad y eliminar, el total calculado
y el botón de checkout.

```typescript
// src/presentation/screens/cart/CartScreen.tsx

import React from 'react';
import {
  View,
  Text,
  FlatList,
  Image,
  TouchableOpacity,
  StyleSheet,
  Alert,
} from 'react-native';
import type { NativeStackScreenProps } from '@react-navigation/native-stack';

import { useCartStore }  from '../../../domain/store/cart.store';
import { Colors }        from '../../../core/theme/colors';
import type { CartItem } from '../../../domain/model/cart.types';
import type { MainTabParamList } from '../../../navigation/MainNavigator';

type Props = NativeStackScreenProps<MainTabParamList, 'CartTab'>;

export default function CartScreen({ navigation }: Props) {
  const { items, total, itemCount, removeItem, updateQuantity, clearCart } = useCartStore();

  // ── Confirmación para vaciar el carrito ─────────────────────────────────────
  const handleClearCart = () => {
    Alert.alert(
      'Vaciar carrito',
      '¿Seguro que quieres eliminar todos los productos del carrito?',
      [
        { text: 'Cancelar', style: 'cancel' },
        { text: 'Vaciar', style: 'destructive', onPress: clearCart },
      ],
    );
  };

  const handleCheckout = () => {
    // Placeholder — se implementará en el módulo de pedidos
    Alert.alert(
      'Proceder al pago',
      `Total: $${total}\n\nLa pasarela de pago se implementa en el próximo módulo.`,
    );
  };

  // ── Render de un ítem ────────────────────────────────────────────────────────
  const renderItem = ({ item }: { item: CartItem }) => {
    const { product, quantity } = item;
    const subtotal = (parseFloat(product.price) * quantity).toFixed(2);

    return (
      <View style={styles.cartItem}>
        {/* Imagen del producto */}
        {product.image_url ? (
          <Image
            source={{ uri: product.image_url }}
            style={styles.itemImage}
            resizeMode="cover"
          />
        ) : (
          <View style={[styles.itemImage, styles.imagePlaceholder]}>
            <Text style={styles.placeholderChar}>?</Text>
          </View>
        )}

        {/* Información */}
        <View style={styles.itemInfo}>
          <Text style={styles.itemName} numberOfLines={2}>
            {product.name}
          </Text>
          <Text style={styles.itemCategory}>{product.category.name}</Text>
          <Text style={styles.itemPrice}>${parseFloat(product.price).toFixed(2)} c/u</Text>

          {/* Controles de cantidad */}
          <View style={styles.qtyRow}>
            <TouchableOpacity
              style={styles.qtyBtn}
              onPress={() => updateQuantity(product.id, quantity - 1)}
            >
              <Text style={styles.qtyBtnText}>−</Text>
            </TouchableOpacity>

            <Text style={styles.qtyText}>{quantity}</Text>

            <TouchableOpacity
              style={styles.qtyBtn}
              onPress={() => updateQuantity(product.id, quantity + 1)}
              disabled={quantity >= product.stock}
            >
              <Text style={styles.qtyBtnText}>+</Text>
            </TouchableOpacity>

            <Text style={styles.subtotal}>${subtotal}</Text>
          </View>
        </View>

        {/* Botón de eliminar */}
        <TouchableOpacity
          style={styles.deleteBtn}
          onPress={() => removeItem(product.id)}
          hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}
        >
          <Text style={styles.deleteBtnText}>✕</Text>
        </TouchableOpacity>
      </View>
    );
  };

  // ── Carrito vacío ─────────────────────────────────────────────────────────
  if (items.length === 0) {
    return (
      <View style={styles.emptyContainer}>
        <Text style={styles.emptyIcon}>🛒</Text>
        <Text style={styles.emptyTitle}>Tu carrito está vacío</Text>
        <Text style={styles.emptySubtitle}>
          Explora el catálogo y agrega productos a tu carrito.
        </Text>
        <TouchableOpacity
          style={styles.goShoppingBtn}
          onPress={() => navigation.navigate('CatalogTab' as any)}
        >
          <Text style={styles.goShoppingText}>Ver catálogo</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <View style={styles.container}>

      {/* ── Header ─────────────────────────────────────────────────────────── */}
      <View style={styles.header}>
        <Text style={styles.headerTitle}>
          Carrito ({itemCount} {itemCount === 1 ? 'producto' : 'productos'})
        </Text>
        <TouchableOpacity onPress={handleClearCart}>
          <Text style={styles.clearText}>Vaciar</Text>
        </TouchableOpacity>
      </View>

      {/* ── Lista de ítems ──────────────────────────────────────────────────── */}
      <FlatList
        data={items}
        keyExtractor={(item) => String(item.product.id)}
        renderItem={renderItem}
        contentContainerStyle={styles.listContent}
        showsVerticalScrollIndicator={false}
        ItemSeparatorComponent={() => <View style={styles.separator} />}
      />

      {/* ── Resumen y checkout ─────────────────────────────────────────────── */}
      <View style={styles.footer}>
        <View style={styles.totalRow}>
          <Text style={styles.totalLabel}>Total</Text>
          <Text style={styles.totalValue}>${total}</Text>
        </View>

        <TouchableOpacity
          style={styles.checkoutBtn}
          onPress={handleCheckout}
          activeOpacity={0.85}
        >
          <Text style={styles.checkoutText}>
            Proceder al pago — ${total}
          </Text>
        </TouchableOpacity>
      </View>

    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex:            1,
    backgroundColor: Colors.background,
  },

  // ── Header
  header: {
    flexDirection:  'row',
    alignItems:     'center',
    justifyContent: 'space-between',
    paddingHorizontal: 16,
    paddingTop:     16,
    paddingBottom:  12,
    borderBottomWidth: 1,
    borderBottomColor: Colors.border,
  },
  headerTitle: {
    color:      Colors.textPrimary,
    fontSize:   20,
    fontWeight: '800',
  },
  clearText: {
    color:     Colors.error,
    fontSize:  14,
    fontWeight: '600',
  },

  // ── Lista
  listContent: {
    padding: 16,
  },
  separator: {
    height:          1,
    backgroundColor: Colors.border,
    marginVertical:  12,
  },

  // ── Ítem
  cartItem: {
    flexDirection:   'row',
    backgroundColor: Colors.surface,
    borderRadius:    12,
    padding:         12,
    gap:             12,
    borderWidth:     1,
    borderColor:     Colors.border,
  },
  itemImage: {
    width:        80,
    height:       80,
    borderRadius: 8,
  },
  imagePlaceholder: {
    backgroundColor: Colors.surface2,
    justifyContent:  'center',
    alignItems:      'center',
  },
  placeholderChar: {
    color:    Colors.textSecondary,
    fontSize: 24,
  },
  itemInfo: {
    flex: 1,
    gap:  3,
  },
  itemName: {
    color:      Colors.textPrimary,
    fontSize:   14,
    fontWeight: '700',
    lineHeight: 20,
  },
  itemCategory: {
    color:    Colors.textSecondary,
    fontSize: 12,
  },
  itemPrice: {
    color:    Colors.textSecondary,
    fontSize: 13,
  },
  qtyRow: {
    flexDirection: 'row',
    alignItems:    'center',
    gap:           10,
    marginTop:     6,
  },
  qtyBtn: {
    backgroundColor: Colors.surface2,
    width:           30,
    height:          30,
    borderRadius:    6,
    justifyContent:  'center',
    alignItems:      'center',
    borderWidth:     1,
    borderColor:     Colors.borderLight,
  },
  qtyBtnText: {
    color:      Colors.textPrimary,
    fontSize:   18,
    fontWeight: '600',
    lineHeight: 22,
  },
  qtyText: {
    color:      Colors.textPrimary,
    fontSize:   16,
    fontWeight: '700',
    minWidth:   24,
    textAlign:  'center',
  },
  subtotal: {
    color:      Colors.accent,
    fontSize:   14,
    fontWeight: '800',
    marginLeft: 4,
  },
  deleteBtn: {
    padding: 4,
  },
  deleteBtnText: {
    color:    Colors.textSecondary,
    fontSize: 16,
  },

  // ── Footer
  footer: {
    backgroundColor: Colors.surface,
    borderTopWidth:  1,
    borderTopColor:  Colors.border,
    padding:         16,
    gap:             12,
  },
  totalRow: {
    flexDirection:  'row',
    justifyContent: 'space-between',
    alignItems:     'center',
  },
  totalLabel: {
    color:      Colors.textSecondary,
    fontSize:   16,
    fontWeight: '600',
  },
  totalValue: {
    color:      Colors.textPrimary,
    fontSize:   24,
    fontWeight: '900',
  },
  checkoutBtn: {
    backgroundColor: Colors.accent,
    borderRadius:    12,
    paddingVertical: 16,
    alignItems:      'center',
  },
  checkoutText: {
    color:      Colors.textPrimary,
    fontSize:   16,
    fontWeight: '800',
  },

  // ── Carrito vacío
  emptyContainer: {
    flex:            1,
    backgroundColor: Colors.background,
    justifyContent:  'center',
    alignItems:      'center',
    padding:         32,
    gap:             12,
  },
  emptyIcon: {
    fontSize:     64,
    marginBottom: 8,
  },
  emptyTitle: {
    color:      Colors.textPrimary,
    fontSize:   22,
    fontWeight: '800',
    textAlign:  'center',
  },
  emptySubtitle: {
    color:     Colors.textSecondary,
    fontSize:  14,
    textAlign: 'center',
    lineHeight: 22,
  },
  goShoppingBtn: {
    backgroundColor: Colors.accent,
    paddingHorizontal: 28,
    paddingVertical:   14,
    borderRadius:      10,
    marginTop:         8,
  },
  goShoppingText: {
    color:      Colors.textPrimary,
    fontWeight: '700',
    fontSize:   15,
  },
});
```

---

## 5.6 Actualizar MainNavigator

**Archivo:** `src/navigation/MainNavigator.tsx`

Agrega `ProductDetailScreen` al `CatalogStack`, la `CartScreen` como pestaña real y coloca
el `CartIcon` con badge en el header de la pantalla de catálogo.

```typescript
// src/navigation/MainNavigator.tsx  (versión completa Módulo 5)

import React from 'react';
import { StyleSheet, View, Text } from 'react-native';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import type { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { useNavigation } from '@react-navigation/native';

import CatalogScreen       from '../presentation/screens/catalog/CatalogScreen';
import ProductDetailScreen from '../presentation/screens/catalog/ProductDetailScreen';
import CartScreen          from '../presentation/screens/cart/CartScreen';
import CartIcon            from '../presentation/components/CartIcon';
import { Colors }          from '../core/theme/colors';

// ─── Tipos de rutas ───────────────────────────────────────────────────────────

export type CatalogStackParamList = {
  Catalog:       undefined;
  ProductDetail: { productId: number };
};

export type MainTabParamList = {
  CatalogTab: undefined;
  CartTab:    undefined;
};

// ─── Navegadores ──────────────────────────────────────────────────────────────

const CatalogStack = createNativeStackNavigator<CatalogStackParamList>();
const Tab          = createBottomTabNavigator<MainTabParamList>();

/**
 * Stack del catálogo: lista → detalle de producto.
 * El header de CatalogScreen incluye el CartIcon con badge.
 */
function CatalogStackNavigator() {
  return (
    <CatalogStack.Navigator
      screenOptions={{
        headerStyle:      { backgroundColor: Colors.surface },
        headerTintColor:  Colors.textPrimary,
        headerTitleStyle: { fontWeight: '700' },
        contentStyle:     { backgroundColor: Colors.background },
      }}
    >
      {/* Pantalla: lista de productos */}
      <CatalogStack.Screen
        name="Catalog"
        component={CatalogScreen}
        options={({ navigation }) => ({
          title: 'Catálogo',
          headerShown: true,
          headerRight: () => (
            <CartIcon onPress={() => navigation.getParent()?.navigate('CartTab')} />
          ),
        })}
      />

      {/* Pantalla: detalle de producto */}
      <CatalogStack.Screen
        name="ProductDetail"
        component={ProductDetailScreen}
        options={({ navigation }) => ({
          title: 'Detalle',
          headerBackTitle: '',
          headerRight: () => (
            <CartIcon onPress={() => navigation.getParent()?.navigate('CartTab')} />
          ),
        })}
      />
    </CatalogStack.Navigator>
  );
}

// ─── MainNavigator (Bottom Tabs) ──────────────────────────────────────────────

export default function MainNavigator() {
  return (
    <Tab.Navigator
      screenOptions={{
        headerShown:             false,
        tabBarStyle:             styles.tabBar,
        tabBarActiveTintColor:   Colors.accent,
        tabBarInactiveTintColor: Colors.textSecondary,
        tabBarLabelStyle:        styles.tabBarLabel,
      }}
    >
      {/* Pestaña Catálogo */}
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

      {/* Pestaña Carrito */}
      <Tab.Screen
        name="CartTab"
        component={CartScreen}
        options={({ navigation }) => ({
          tabBarLabel: 'Carrito',
          tabBarIcon: ({ color }) => (
            <Text style={{ color, fontSize: 20 }}>🛒</Text>
          ),
          // Mostramos el header estándar en la pantalla de carrito
          headerShown:  true,
          headerStyle:  { backgroundColor: Colors.surface },
          headerTintColor: Colors.textPrimary,
          headerTitleStyle: { fontWeight: '700' },
          title: 'Mi carrito',
        })}
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
});
```

> **`navigation.getParent()?.navigate('CartTab')`**
> Cuando el `CartIcon` está dentro del `CatalogStack`, `navigation` apunta al stack hijo.
> Usamos `getParent()` para subir al `Tab` navigator y navegar a la pestaña `CartTab`.

---

## Estructura de navegación completa (Módulos 3–5)

```
NavigationContainer
  AppNavigator
    ├── AuthNavigator  (si user === null)
    │     ├── LoginScreen
    │     └── RegisterScreen
    └── MainNavigator  (si user !== null)
          ├── Tab: CatalogTab
          │     └── CatalogStack
          │           ├── CatalogScreen       (lista + búsqueda + filtros)
          │           └── ProductDetailScreen (detalle + agregar al carrito)
          └── Tab: CartTab
                └── CartScreen              (lista de ítems + checkout)
```

---

## Resumen de archivos del Módulo 5

| Archivo | Descripción |
|---|---|
| `src/domain/model/cart.types.ts` | Interfaces CartItem y CartState |
| `src/domain/store/cart.store.ts` | Store Zustand del carrito con cálculos reactivos |
| `src/presentation/screens/catalog/ProductDetailScreen.tsx` | Detalle completo del producto |
| `src/presentation/components/CartIcon.tsx` | Ícono de carrito con badge reactivo |
| `src/presentation/screens/cart/CartScreen.tsx` | Pantalla del carrito con controles y total |
| `src/navigation/MainNavigator.tsx` | Bottom tabs con ProductDetail y CartScreen integrados |

---

## Checkpoint final — Módulo 5

**Prueba 1: Navegar al detalle de producto**
1. Toca cualquier tarjeta en el catálogo.
2. Debe abrirse `ProductDetailScreen` con imagen grande, nombre, precio base, precio con IVA,
   descripción y estado de stock.
3. Si el producto tiene `in_stock: false`, el botón "Agregar al carrito" no debe aparecer.

**Prueba 2: Agregar al carrito**
1. En la pantalla de detalle, ajusta la cantidad con los botones + y −.
2. Presiona "Agregar al carrito".
3. El badge del ícono del carrito en el header debe actualizarse inmediatamente.
4. El diálogo debe ofrecer "Seguir comprando" o "Ver carrito".

**Prueba 3: Respeto del stock**
1. Intenta agregar más unidades de las que hay en stock.
2. Debe aparecer un Alert informando el límite.
3. El botón + debe quedar deshabilitado al llegar al máximo.

**Prueba 4: Pantalla de carrito**
1. Navega a la pestaña Carrito (o desde el diálogo tras agregar).
2. Deben listarse todos los ítems con imagen, nombre, precio unitario, subtotal y controles.
3. Al presionar − en un ítem con cantidad 1, el ítem se elimina.
4. El total al pie de la pantalla se recalcula en cada cambio.

**Prueba 5: Vaciar carrito**
1. En la pantalla de carrito presiona "Vaciar".
2. Confirma en el Alert.
3. Debe mostrarse el estado vacío con el botón "Ver catálogo".

**Prueba 6: Badge en header**
1. Agrega varios productos distintos.
2. El badge debe mostrar el total de unidades (no de productos únicos).
3. Al eliminar todos los ítems el badge desaparece.
