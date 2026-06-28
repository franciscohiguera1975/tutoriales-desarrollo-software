# ShopApp React Native — Módulo 6

## Pedidos — Checkout, confirmación y estado

**Objetivo:** Implementar el flujo completo de compra: desde el carrito hasta la confirmación del pedido. El usuario podrá crear un pedido, añadir los productos del carrito, confirmar la compra y ver el historial de sus pedidos.

> **Checkpoint final:** El usuario puede completar una compra desde el carrito, ver la pantalla de éxito con el ID del pedido y listar todos sus pedidos anteriores con su estado correspondiente.

---

## 6.1 Tipos de pedidos

**Archivo:** `src/domain/model/orders.types.ts`

Define las interfaces TypeScript que representan la estructura de datos que devuelve la API de pedidos.

```typescript
// src/domain/model/orders.types.ts

export interface OrderItem {
  id: number;
  product: {
    id: number;
    name: string;
    price: string;
    image_url: string | null;
  };
  quantity: number;
  unit_price: string;
  subtotal: string;
}

export interface Order {
  id: number;
  status: 'pending' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';
  total: string;
  created_at: string;
  updated_at: string;
  items: OrderItem[];
}

export interface PaginatedOrders {
  count: number;
  next: string | null;
  previous: string | null;
  results: Order[];
}

// Parámetros para listar pedidos
export interface OrderListParams {
  page?: number;
  status?: Order['status'];
}

// Payload para añadir un ítem
export interface AddItemPayload {
  product_id: number;
  quantity: number;
}

// Payload para actualizar estado (uso admin, módulo 9)
export interface UpdateStatusPayload {
  status: Order['status'];
}
```

**Puntos clave:**
- `status` usa un union type para restringir los valores posibles.
- `total`, `unit_price` y `subtotal` vienen como `string` desde Django REST Framework (campo `DecimalField`); conviértelos con `parseFloat()` cuando necesites operar con ellos.
- `PaginatedOrders` refleja la estructura de paginación de DRF con `count`, `next`, `previous` y `results`.

---

## 6.2 OrdersStore

**Archivo:** `src/domain/store/orders.store.ts`

Store de Zustand que maneja el estado global de pedidos y expone las acciones que consumen la API.

```typescript
// src/domain/store/orders.store.ts

import { create } from 'zustand';
import { Order, PaginatedOrders, OrderListParams } from '../model/orders.types';
import { apiClient } from '../../../services/api/apiClient';

interface OrdersState {
  // Estado
  orders: Order[];
  currentOrder: Order | null;
  isLoading: boolean;
  error: string | null;
  totalCount: number;
  currentPage: number;
  hasNextPage: boolean;

  // Acciones
  loadOrders: (params?: OrderListParams) => Promise<void>;
  loadMoreOrders: () => Promise<void>;
  createOrder: () => Promise<Order>;
  addItemToOrder: (orderId: number, productId: number, quantity: number) => Promise<void>;
  confirmCurrentOrder: () => Promise<void>;
  setCurrentOrder: (order: Order | null) => void;
  clearError: () => void;
  resetStore: () => void;
}

const initialState = {
  orders: [],
  currentOrder: null,
  isLoading: false,
  error: null,
  totalCount: 0,
  currentPage: 1,
  hasNextPage: false,
};

export const useOrdersStore = create<OrdersState>((set, get) => ({
  ...initialState,

  // Carga la lista paginada de pedidos del usuario autenticado
  loadOrders: async (params?: OrderListParams) => {
    set({ isLoading: true, error: null });
    try {
      const response = await apiClient.get<PaginatedOrders>('/api/orders/', {
        params: { page: 1, ...params },
      });
      set({
        orders: response.data.results,
        totalCount: response.data.count,
        currentPage: 1,
        hasNextPage: response.data.next !== null,
        isLoading: false,
      });
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al cargar los pedidos',
        isLoading: false,
      });
    }
  },

  // Carga la siguiente página y agrega resultados al array existente
  loadMoreOrders: async () => {
    const { isLoading, hasNextPage, currentPage, orders } = get();
    if (isLoading || !hasNextPage) return;

    set({ isLoading: true });
    try {
      const nextPage = currentPage + 1;
      const response = await apiClient.get<PaginatedOrders>('/api/orders/', {
        params: { page: nextPage },
      });
      set({
        orders: [...orders, ...response.data.results],
        currentPage: nextPage,
        hasNextPage: response.data.next !== null,
        isLoading: false,
      });
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al cargar más pedidos',
        isLoading: false,
      });
    }
  },

  // Crea un pedido vacío y lo guarda como pedido activo
  createOrder: async () => {
    const response = await apiClient.post<Order>('/api/orders/');
    const newOrder = response.data;
    set({ currentOrder: newOrder });
    return newOrder;
  },

  // Añade un ítem al pedido dado
  addItemToOrder: async (orderId: number, productId: number, quantity: number) => {
    await apiClient.post(`/api/orders/${orderId}/add-item/`, {
      product_id: productId,
      quantity,
    });
  },

  // Confirma el pedido activo actual
  confirmCurrentOrder: async () => {
    const { currentOrder } = get();
    if (!currentOrder) throw new Error('No hay pedido activo para confirmar');

    const response = await apiClient.post<Order>(
      `/api/orders/${currentOrder.id}/confirm/`,
    );
    set({ currentOrder: response.data });
  },

  setCurrentOrder: (order) => set({ currentOrder: order }),
  clearError: () => set({ error: null }),

  resetStore: () => set(initialState),
}));
```

**Puntos clave:**
- `createOrder` lanza la excepción si falla; el caller (`CheckoutScreen`) la captura para mostrar el error al usuario.
- `loadMoreOrders` verifica `hasNextPage` antes de hacer la petición para evitar llamadas innecesarias.
- `resetStore` es útil al hacer logout para limpiar el estado.

---

## 6.3 CheckoutScreen

**Archivo:** `src/presentation/screens/orders/CheckoutScreen.tsx`

Pantalla que resume los productos del carrito y ejecuta el flujo completo de creación y confirmación del pedido.

```typescript
// src/presentation/screens/orders/CheckoutScreen.tsx

import React, { useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  ScrollView,
  TouchableOpacity,
  ActivityIndicator,
  Alert,
  Image,
} from 'react-native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { useNavigation } from '@react-navigation/native';
import { Colors } from '../../../constants/Colors';
import { useCartStore } from '../../cart/store/cart.store';
import { useOrdersStore } from '../../../domain/store/orders.store';
import { RootStackParamList } from '../../../presentation/navigation/types';

type CheckoutNav = NativeStackNavigationProp<RootStackParamList, 'Checkout'>;

type CheckoutStep =
  | 'idle'
  | 'creating'
  | 'adding_items'
  | 'confirming'
  | 'done';

function stepLabel(step: CheckoutStep): string {
  switch (step) {
    case 'creating':
      return 'Creando pedido…';
    case 'adding_items':
      return 'Añadiendo productos…';
    case 'confirming':
      return 'Confirmando pedido…';
    default:
      return '';
  }
}

export default function CheckoutScreen() {
  const navigation = useNavigation<CheckoutNav>();
  const { items, totalPrice, clearCart } = useCartStore();
  const { createOrder, addItemToOrder, confirmCurrentOrder } = useOrdersStore();

  const [step, setStep] = useState<CheckoutStep>('idle');
  const isProcessing = step !== 'idle' && step !== 'done';

  const handleConfirmOrder = async () => {
    if (items.length === 0) {
      Alert.alert('Carrito vacío', 'Añade productos antes de confirmar.');
      return;
    }

    try {
      // Paso 1: Crear el pedido vacío
      setStep('creating');
      const order = await createOrder();

      // Paso 2: Añadir cada ítem del carrito
      setStep('adding_items');
      for (const item of items) {
        await addItemToOrder(order.id, item.product.id, item.quantity);
      }

      // Paso 3: Confirmar el pedido
      setStep('confirming');
      await confirmCurrentOrder();

      // Paso 4: Limpiar el carrito
      clearCart();
      setStep('done');

      // Paso 5: Navegar a la pantalla de éxito
      navigation.replace('OrderSuccess', {
        orderId: order.id,
        total: totalPrice.toFixed(2),
      });
    } catch (err: any) {
      setStep('idle');
      const message =
        err.response?.data?.detail ?? err.message ?? 'Ocurrió un error inesperado';
      Alert.alert('Error al procesar el pedido', message);
    }
  };

  return (
    <View style={styles.container}>
      <ScrollView contentContainerStyle={styles.scroll}>
        <Text style={styles.title}>Resumen del Pedido</Text>

        {/* Lista de ítems del carrito */}
        {items.map((item) => (
          <View key={item.product.id} style={styles.itemRow}>
            {item.product.image_url ? (
              <Image
                source={{ uri: item.product.image_url }}
                style={styles.itemImage}
              />
            ) : (
              <View style={[styles.itemImage, styles.imagePlaceholder]} />
            )}
            <View style={styles.itemInfo}>
              <Text style={styles.itemName} numberOfLines={2}>
                {item.product.name}
              </Text>
              <Text style={styles.itemQty}>
                {item.quantity} × ${parseFloat(item.product.price).toFixed(2)}
              </Text>
            </View>
            <Text style={styles.itemSubtotal}>
              ${(item.quantity * parseFloat(item.product.price)).toFixed(2)}
            </Text>
          </View>
        ))}

        {/* Separador y total */}
        <View style={styles.divider} />
        <View style={styles.totalRow}>
          <Text style={styles.totalLabel}>Total</Text>
          <Text style={styles.totalValue}>${totalPrice.toFixed(2)}</Text>
        </View>
      </ScrollView>

      {/* Indicador de progreso */}
      {isProcessing && (
        <View style={styles.loadingOverlay}>
          <ActivityIndicator size="large" color={Colors.accent} />
          <Text style={styles.loadingText}>{stepLabel(step)}</Text>
        </View>
      )}

      {/* Botón de confirmación */}
      <View style={styles.footer}>
        <TouchableOpacity
          style={[styles.confirmBtn, isProcessing && styles.confirmBtnDisabled]}
          onPress={handleConfirmOrder}
          disabled={isProcessing}
          activeOpacity={0.8}
        >
          {isProcessing ? (
            <ActivityIndicator color={Colors.textPrimary} />
          ) : (
            <Text style={styles.confirmBtnText}>Confirmar Pedido</Text>
          )}
        </TouchableOpacity>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  scroll: {
    padding: 20,
    paddingBottom: 120,
  },
  title: {
    fontSize: 22,
    fontWeight: '700',
    color: Colors.textPrimary,
    marginBottom: 20,
  },
  itemRow: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: Colors.surface,
    borderRadius: 12,
    padding: 12,
    marginBottom: 10,
    gap: 12,
  },
  itemImage: {
    width: 56,
    height: 56,
    borderRadius: 8,
  },
  imagePlaceholder: {
    backgroundColor: Colors.surface2,
  },
  itemInfo: {
    flex: 1,
  },
  itemName: {
    fontSize: 14,
    fontWeight: '600',
    color: Colors.textPrimary,
    marginBottom: 4,
  },
  itemQty: {
    fontSize: 12,
    color: Colors.textSecondary,
  },
  itemSubtotal: {
    fontSize: 14,
    fontWeight: '700',
    color: Colors.accent,
  },
  divider: {
    height: 1,
    backgroundColor: Colors.border,
    marginVertical: 16,
  },
  totalRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  totalLabel: {
    fontSize: 18,
    fontWeight: '600',
    color: Colors.textSecondary,
  },
  totalValue: {
    fontSize: 22,
    fontWeight: '800',
    color: Colors.textPrimary,
  },
  loadingOverlay: {
    position: 'absolute',
    inset: 0,
    backgroundColor: 'rgba(15,17,23,0.75)',
    alignItems: 'center',
    justifyContent: 'center',
    gap: 12,
  },
  loadingText: {
    fontSize: 16,
    color: Colors.textPrimary,
    fontWeight: '500',
  },
  footer: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    padding: 20,
    backgroundColor: Colors.background,
    borderTopWidth: 1,
    borderTopColor: Colors.border,
  },
  confirmBtn: {
    backgroundColor: Colors.accent,
    borderRadius: 14,
    paddingVertical: 16,
    alignItems: 'center',
  },
  confirmBtnDisabled: {
    opacity: 0.6,
  },
  confirmBtnText: {
    color: Colors.textPrimary,
    fontSize: 16,
    fontWeight: '700',
  },
});
```

**Flujo detallado:**

| Paso | Acción | Estado del indicador |
|------|--------|----------------------|
| 1 | `createOrder()` → POST /api/orders/ | `creating` |
| 2 | `addItemToOrder()` por cada ítem | `adding_items` |
| 3 | `confirmCurrentOrder()` → POST /api/orders/{id}/confirm/ | `confirming` |
| 4 | `clearCart()` — limpia el store local | — |
| 5 | `navigation.replace('OrderSuccess', …)` | — |

> Usamos `navigation.replace` en lugar de `navigate` para que el usuario no pueda volver a Checkout con el botón atrás después de confirmar.

---

## 6.4 OrderSuccessScreen

**Archivo:** `src/presentation/screens/orders/OrderSuccessScreen.tsx`

Pantalla de confirmación que muestra al usuario que su pedido fue procesado con éxito.

```typescript
// src/presentation/screens/orders/OrderSuccessScreen.tsx

import React from 'react';
import {
  View,
  Text,
  StyleSheet,
  TouchableOpacity,
} from 'react-native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { RouteProp, useNavigation, useRoute } from '@react-navigation/native';
import { Colors } from '../../../constants/Colors';
import { RootStackParamList } from '../../../presentation/navigation/types';

type SuccessRoute = RouteProp<RootStackParamList, 'OrderSuccess'>;
type SuccessNav = NativeStackNavigationProp<RootStackParamList, 'OrderSuccess'>;

export default function OrderSuccessScreen() {
  const navigation = useNavigation<SuccessNav>();
  const route = useRoute<SuccessRoute>();
  const { orderId, total } = route.params;

  const handleViewOrders = () => {
    // Navega al tab de Pedidos y resetea el stack de ese tab
    navigation.reset({
      index: 0,
      routes: [{ name: 'MainTabs', params: { screen: 'Orders' } }],
    });
  };

  const handleKeepShopping = () => {
    // Regresa al tab de Home
    navigation.reset({
      index: 0,
      routes: [{ name: 'MainTabs', params: { screen: 'Home' } }],
    });
  };

  return (
    <View style={styles.container}>
      {/* Ícono de éxito */}
      <View style={styles.iconContainer}>
        <Text style={styles.checkIcon}>✓</Text>
      </View>

      <Text style={styles.title}>¡Pedido confirmado!</Text>
      <Text style={styles.subtitle}>
        Tu pedido ha sido procesado con éxito.
      </Text>

      {/* Detalles del pedido */}
      <View style={styles.detailCard}>
        <View style={styles.detailRow}>
          <Text style={styles.detailLabel}>Número de pedido</Text>
          <Text style={styles.detailValue}>#{orderId}</Text>
        </View>
        <View style={styles.divider} />
        <View style={styles.detailRow}>
          <Text style={styles.detailLabel}>Total</Text>
          <Text style={[styles.detailValue, styles.totalValue]}>${total}</Text>
        </View>
      </View>

      <Text style={styles.infoText}>
        Recibirás actualizaciones sobre el estado de tu pedido en la sección
        "Mis Pedidos".
      </Text>

      {/* Acciones */}
      <View style={styles.actions}>
        <TouchableOpacity
          style={styles.primaryBtn}
          onPress={handleViewOrders}
          activeOpacity={0.8}
        >
          <Text style={styles.primaryBtnText}>Ver mis pedidos</Text>
        </TouchableOpacity>

        <TouchableOpacity
          style={styles.secondaryBtn}
          onPress={handleKeepShopping}
          activeOpacity={0.8}
        >
          <Text style={styles.secondaryBtnText}>Seguir comprando</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 24,
  },
  iconContainer: {
    width: 100,
    height: 100,
    borderRadius: 50,
    backgroundColor: Colors.success,
    alignItems: 'center',
    justifyContent: 'center',
    marginBottom: 24,
    // Sombra sutil
    shadowColor: Colors.success,
    shadowOpacity: 0.4,
    shadowRadius: 20,
    shadowOffset: { width: 0, height: 8 },
    elevation: 10,
  },
  checkIcon: {
    fontSize: 48,
    color: Colors.textPrimary,
    fontWeight: '900',
  },
  title: {
    fontSize: 28,
    fontWeight: '800',
    color: Colors.textPrimary,
    marginBottom: 8,
    textAlign: 'center',
  },
  subtitle: {
    fontSize: 16,
    color: Colors.textSecondary,
    textAlign: 'center',
    marginBottom: 32,
  },
  detailCard: {
    backgroundColor: Colors.surface,
    borderRadius: 16,
    padding: 20,
    width: '100%',
    marginBottom: 20,
  },
  detailRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 6,
  },
  detailLabel: {
    fontSize: 14,
    color: Colors.textSecondary,
  },
  detailValue: {
    fontSize: 15,
    fontWeight: '600',
    color: Colors.textPrimary,
  },
  totalValue: {
    fontSize: 20,
    fontWeight: '800',
    color: Colors.accent,
  },
  divider: {
    height: 1,
    backgroundColor: Colors.border,
    marginVertical: 10,
  },
  infoText: {
    fontSize: 13,
    color: Colors.textSecondary,
    textAlign: 'center',
    marginBottom: 36,
    lineHeight: 20,
    paddingHorizontal: 8,
  },
  actions: {
    width: '100%',
    gap: 12,
  },
  primaryBtn: {
    backgroundColor: Colors.accent,
    borderRadius: 14,
    paddingVertical: 16,
    alignItems: 'center',
  },
  primaryBtnText: {
    color: Colors.textPrimary,
    fontSize: 16,
    fontWeight: '700',
  },
  secondaryBtn: {
    backgroundColor: Colors.surface,
    borderRadius: 14,
    paddingVertical: 16,
    alignItems: 'center',
    borderWidth: 1,
    borderColor: Colors.border,
  },
  secondaryBtnText: {
    color: Colors.textPrimary,
    fontSize: 16,
    fontWeight: '600',
  },
});
```

> **Nota sobre los parámetros de ruta:** Asegúrate de definir `OrderSuccess` en `RootStackParamList`:
> ```typescript
> OrderSuccess: { orderId: number; total: string };
> ```

---

## 6.5 OrdersScreen

**Archivo:** `src/presentation/screens/orders/OrdersScreen.tsx`

Lista todos los pedidos del usuario autenticado. Cada ítem muestra el ID, estado con badge de color, total y fecha. Se puede tocar para ver el detalle (módulo 7).

```typescript
// src/presentation/screens/orders/OrdersScreen.tsx

import React, { useEffect, useCallback } from 'react';
import {
  View,
  Text,
  StyleSheet,
  FlatList,
  TouchableOpacity,
  ActivityIndicator,
  RefreshControl,
  ListRenderItemInfo,
} from 'react-native';
import { useNavigation } from '@react-navigation/native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { Colors } from '../../../constants/Colors';
import { useOrdersStore } from '../../../domain/store/orders.store';
import { Order } from '../../../domain/model/orders.types';
import { RootStackParamList } from '../../../presentation/navigation/types';

type OrdersNav = NativeStackNavigationProp<RootStackParamList>;

// Mapa de colores para cada estado
const STATUS_CONFIG: Record<
  string,
  { label: string; bg: string; text: string }
> = {
  pending:   { label: 'Pendiente',   bg: '#FFA000',  text: '#FFFFFF' },
  confirmed: { label: 'Confirmado',  bg: '#1976D2',  text: '#FFFFFF' },
  shipped:   { label: 'Enviado',     bg: '#7B1FA2',  text: '#FFFFFF' },
  delivered: { label: 'Entregado',   bg: Colors.success, text: '#FFFFFF' },
  cancelled: { label: 'Cancelado',   bg: Colors.error,   text: '#FFFFFF' },
};

function StatusBadgeInline({ status }: { status: string }) {
  const config = STATUS_CONFIG[status] ?? {
    label: status,
    bg: Colors.surface2,
    text: Colors.textPrimary,
  };
  return (
    <View style={[badgeStyles.badge, { backgroundColor: config.bg }]}>
      <Text style={[badgeStyles.text, { color: config.text }]}>
        {config.label}
      </Text>
    </View>
  );
}

const badgeStyles = StyleSheet.create({
  badge: {
    borderRadius: 8,
    paddingHorizontal: 10,
    paddingVertical: 3,
  },
  text: {
    fontSize: 11,
    fontWeight: '700',
    textTransform: 'uppercase',
    letterSpacing: 0.5,
  },
});

function formatDate(iso: string): string {
  const date = new Date(iso);
  return date.toLocaleDateString('es-ES', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  });
}

function OrderCard({
  order,
  onPress,
}: {
  order: Order;
  onPress: () => void;
}) {
  return (
    <TouchableOpacity
      style={styles.card}
      onPress={onPress}
      activeOpacity={0.75}
    >
      <View style={styles.cardHeader}>
        <Text style={styles.orderId}>Pedido #{order.id}</Text>
        <StatusBadgeInline status={order.status} />
      </View>
      <View style={styles.cardBody}>
        <Text style={styles.itemCount}>
          {order.items.length}{' '}
          {order.items.length === 1 ? 'producto' : 'productos'}
        </Text>
        <Text style={styles.date}>{formatDate(order.created_at)}</Text>
      </View>
      <View style={styles.cardFooter}>
        <Text style={styles.totalLabel}>Total</Text>
        <Text style={styles.total}>${parseFloat(order.total).toFixed(2)}</Text>
      </View>
    </TouchableOpacity>
  );
}

export default function OrdersScreen() {
  const navigation = useNavigation<OrdersNav>();
  const {
    orders,
    isLoading,
    error,
    hasNextPage,
    loadOrders,
    loadMoreOrders,
  } = useOrdersStore();

  useEffect(() => {
    loadOrders();
  }, []);

  const handleRefresh = useCallback(() => {
    loadOrders();
  }, []);

  const handleLoadMore = useCallback(() => {
    if (hasNextPage && !isLoading) {
      loadMoreOrders();
    }
  }, [hasNextPage, isLoading]);

  const handlePressOrder = (order: Order) => {
    navigation.navigate('OrderDetail', { orderId: order.id });
  };

  const renderItem = ({ item }: ListRenderItemInfo<Order>) => (
    <OrderCard order={item} onPress={() => handlePressOrder(item)} />
  );

  const renderFooter = () => {
    if (!hasNextPage) return null;
    return (
      <View style={styles.footerLoader}>
        <ActivityIndicator color={Colors.accent} />
      </View>
    );
  };

  if (isLoading && orders.length === 0) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color={Colors.accent} />
      </View>
    );
  }

  if (error && orders.length === 0) {
    return (
      <View style={styles.center}>
        <Text style={styles.errorText}>{error}</Text>
        <TouchableOpacity style={styles.retryBtn} onPress={() => loadOrders()}>
          <Text style={styles.retryText}>Reintentar</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <FlatList
        data={orders}
        keyExtractor={(item) => String(item.id)}
        renderItem={renderItem}
        contentContainerStyle={styles.list}
        ListHeaderComponent={
          <Text style={styles.screenTitle}>Mis Pedidos</Text>
        }
        ListEmptyComponent={
          <View style={styles.empty}>
            <Text style={styles.emptyText}>
              Aún no tienes pedidos.{'\n'}¡Empieza a comprar!
            </Text>
          </View>
        }
        ListFooterComponent={renderFooter}
        refreshControl={
          <RefreshControl
            refreshing={isLoading && orders.length > 0}
            onRefresh={handleRefresh}
            tintColor={Colors.accent}
          />
        }
        onEndReached={handleLoadMore}
        onEndReachedThreshold={0.4}
        showsVerticalScrollIndicator={false}
      />
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  list: {
    padding: 16,
    paddingBottom: 32,
  },
  screenTitle: {
    fontSize: 24,
    fontWeight: '800',
    color: Colors.textPrimary,
    marginBottom: 16,
  },
  card: {
    backgroundColor: Colors.surface,
    borderRadius: 14,
    padding: 16,
    marginBottom: 12,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  cardHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 10,
  },
  orderId: {
    fontSize: 15,
    fontWeight: '700',
    color: Colors.textPrimary,
  },
  cardBody: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginBottom: 10,
  },
  itemCount: {
    fontSize: 13,
    color: Colors.textSecondary,
  },
  date: {
    fontSize: 13,
    color: Colors.textSecondary,
  },
  cardFooter: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    borderTopWidth: 1,
    borderTopColor: Colors.border,
    paddingTop: 10,
    marginTop: 4,
  },
  totalLabel: {
    fontSize: 13,
    color: Colors.textSecondary,
  },
  total: {
    fontSize: 16,
    fontWeight: '800',
    color: Colors.accent,
  },
  center: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: Colors.background,
  },
  errorText: {
    color: Colors.error,
    fontSize: 15,
    textAlign: 'center',
    marginBottom: 16,
  },
  retryBtn: {
    backgroundColor: Colors.accent,
    borderRadius: 10,
    paddingHorizontal: 20,
    paddingVertical: 10,
  },
  retryText: {
    color: Colors.textPrimary,
    fontWeight: '600',
  },
  empty: {
    alignItems: 'center',
    paddingTop: 60,
  },
  emptyText: {
    fontSize: 16,
    color: Colors.textSecondary,
    textAlign: 'center',
    lineHeight: 24,
  },
  footerLoader: {
    paddingVertical: 16,
    alignItems: 'center',
  },
});
```

---

## 6.6 Integrar en MainNavigator

**Archivo:** `src/presentation/navigation/MainNavigator.tsx` _(actualizar)_

Añade la pestaña de **Pedidos** en el bottom tab navigator y expone `CheckoutScreen` y `OrderSuccessScreen` en el stack raíz para que sean accesibles desde el carrito.

```typescript
// src/presentation/navigation/MainNavigator.tsx (fragmento relevante)

import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

// Pantallas existentes
import HomeScreen from '../screens/home/HomeScreen';
import CartScreen from '../screens/cart/CartScreen';

// Nuevas pantallas de pedidos
import OrdersScreen from '../screens/orders/OrdersScreen';
import CheckoutScreen from '../screens/orders/CheckoutScreen';
import OrderSuccessScreen from '../screens/orders/OrderSuccessScreen';
import OrderDetailScreen from '../screens/orders/OrderDetailScreen';

const Tab = createBottomTabNavigator();
const Stack = createNativeStackNavigator();

// Stack raíz que envuelve los tabs y expone pantallas modales/de flujo
function RootStack() {
  return (
    <Stack.Navigator
      screenOptions={{
        headerStyle: { backgroundColor: Colors.surface },
        headerTintColor: Colors.textPrimary,
        headerTitleStyle: { fontWeight: '700' },
        contentStyle: { backgroundColor: Colors.background },
      }}
    >
      {/* Tabs principales */}
      <Stack.Screen
        name="MainTabs"
        component={MainTabs}
        options={{ headerShown: false }}
      />

      {/* Flujo de checkout — accesible desde Cart */}
      <Stack.Screen
        name="Checkout"
        component={CheckoutScreen}
        options={{ title: 'Finalizar Compra' }}
      />
      <Stack.Screen
        name="OrderSuccess"
        component={OrderSuccessScreen}
        options={{ headerShown: false }}
      />

      {/* Detalle de pedido — accesible desde la tab Orders */}
      <Stack.Screen
        name="OrderDetail"
        component={OrderDetailScreen}
        options={{ title: 'Detalle del Pedido' }}
      />
    </Stack.Navigator>
  );
}

// Tabs inferiores
function MainTabs() {
  return (
    <Tab.Navigator
      screenOptions={{
        tabBarStyle: {
          backgroundColor: Colors.surface,
          borderTopColor: Colors.border,
        },
        tabBarActiveTintColor: Colors.accent,
        tabBarInactiveTintColor: Colors.textSecondary,
      }}
    >
      <Tab.Screen
        name="Home"
        component={HomeScreen}
        options={{ title: 'Inicio' }}
      />
      <Tab.Screen
        name="Cart"
        component={CartScreen}
        options={{ title: 'Carrito' }}
      />
      {/* Nueva pestaña de pedidos */}
      <Tab.Screen
        name="Orders"
        component={OrdersScreen}
        options={{ title: 'Pedidos' }}
      />
    </Tab.Navigator>
  );
}

export default RootStack;
```

**Actualiza también `RootStackParamList`:**

```typescript
// src/presentation/navigation/types.ts (añadir)

export type RootStackParamList = {
  MainTabs: { screen?: string } | undefined;
  Checkout: undefined;
  OrderSuccess: { orderId: number; total: string };
  OrderDetail: { orderId: number };
  // ... rest of params
};
```

---

## ✅ Checkpoint — Módulo 6

Verifica que el flujo completo funcione de extremo a extremo:

- [ ] Al presionar "Ir al Checkout" desde `CartScreen`, navega a `CheckoutScreen` y se muestran los productos del carrito con sus precios y el total correcto.
- [ ] Al presionar "Confirmar Pedido" se ejecutan las tres llamadas a la API en orden (crear, añadir ítems, confirmar) con el indicador de paso visible.
- [ ] Tras la confirmación, el carrito queda vacío y la app navega a `OrderSuccessScreen` mostrando el ID del pedido y el total.
- [ ] `OrdersScreen` carga y muestra la lista de pedidos del usuario con el badge de estado en el color correcto.
- [ ] El scroll infinito funciona: al llegar al final de la lista se cargan más pedidos si `hasNextPage` es `true`.
- [ ] El pull-to-refresh recarga la lista desde la página 1.
- [ ] La pestaña "Pedidos" aparece en el bottom tab navigator.

---

**Siguiente módulo:** El módulo 7 amplía esta funcionalidad mostrando el detalle completo de cada pedido con sus ítems, imágenes de producto y timeline de estado.
