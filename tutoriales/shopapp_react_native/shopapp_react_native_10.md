# ShopApp React Native — Módulo 10

## Panel Admin — Gestión de Pedidos

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulos 1–9 completados. El usuario autenticado debe tener `is_staff = true` para acceder a las rutas de este módulo.

---

> **Objetivo**
> Implementar el panel de administración de pedidos: listar todos los pedidos del sistema (no solo los del usuario actual), filtrarlos por estado y actualizar el estado de cada pedido desde la app móvil.

---

> **Checkpoint final**
> Al terminar este módulo deberás poder:
> - Ver la lista completa de pedidos de todos los usuarios
> - Filtrar pedidos por estado (pending, confirmed, shipped, delivered, cancelled)
> - Cambiar el estado de cualquier pedido desde un modal
> - Ver el detalle completo de un pedido como administrador

---

## 10.1 AdminOrdersStore

**Archivo:** `src/domain/store/admin.orders.store.ts`

Este store maneja el estado global de la gestión de pedidos para el panel de administración. Utiliza Zustand con soporte para filtros por estado y paginación.

### Tipos necesarios

Crea el archivo `src/domain/model/admin.orders.types.ts`:

```typescript
export type OrderStatus =
  | 'pending'
  | 'confirmed'
  | 'shipped'
  | 'delivered'
  | 'cancelled';

export interface AdminOrderItem {
  id: number;
  product: number;
  product_name: string;
  product_sku: string;
  quantity: number;
  unit_price: string;
  subtotal: string;
}

export interface AdminOrder {
  id: number;
  user: number;
  user_username: string;
  user_email: string;
  status: OrderStatus;
  total: string;
  created_at: string;
  updated_at: string;
  shipping_address: string;
  items: AdminOrderItem[];
}

export interface AdminOrdersState {
  orders: AdminOrder[];
  selectedOrder: AdminOrder | null;
  statusFilter: OrderStatus | 'all';
  isLoading: boolean;
  isUpdating: boolean;
  error: string | null;
  currentPage: number;
  totalPages: number;
}

export interface AdminOrdersActions {
  loadOrders: (page?: number) => Promise<void>;
  loadOrderDetail: (orderId: number) => Promise<void>;
  updateOrderStatus: (orderId: number, status: OrderStatus) => Promise<void>;
  setStatusFilter: (filter: OrderStatus | 'all') => void;
  resetError: () => void;
}
```

### Implementación del store

```typescript
// src/domain/store/admin.orders.store.ts
import { create } from 'zustand';
import {
  AdminOrdersState,
  AdminOrdersActions,
  AdminOrder,
  OrderStatus,
} from '../model/admin.orders.types';
import { apiClient } from '../../data/api/api.client';

const PAGE_SIZE = 20;

const initialState: AdminOrdersState = {
  orders: [],
  selectedOrder: null,
  statusFilter: 'all',
  isLoading: false,
  isUpdating: false,
  error: null,
  currentPage: 1,
  totalPages: 1,
};

export const useAdminOrdersStore = create<AdminOrdersState & AdminOrdersActions>(
  (set, get) => ({
    ...initialState,

    loadOrders: async (page = 1) => {
      const { statusFilter } = get();
      set({ isLoading: true, error: null });

      try {
        const params: Record<string, string | number> = {
          page,
          page_size: PAGE_SIZE,
        };

        // El backend filtra por status si se pasa el parámetro
        if (statusFilter !== 'all') {
          params.status = statusFilter;
        }

        const response = await apiClient.get<{
          results: AdminOrder[];
          count: number;
        }>('/api/orders/', { params });

        set({
          orders: response.data.results,
          currentPage: page,
          totalPages: Math.ceil(response.data.count / PAGE_SIZE),
          isLoading: false,
        });
      } catch (error: any) {
        set({
          error: error?.response?.data?.detail || 'Error al cargar pedidos',
          isLoading: false,
        });
      }
    },

    loadOrderDetail: async (orderId: number) => {
      set({ isLoading: true, error: null });
      try {
        const response = await apiClient.get<AdminOrder>(
          `/api/orders/${orderId}/`,
        );
        set({ selectedOrder: response.data, isLoading: false });
      } catch (error: any) {
        set({
          error: error?.response?.data?.detail || 'Error al cargar el pedido',
          isLoading: false,
        });
      }
    },

    updateOrderStatus: async (orderId: number, status: OrderStatus) => {
      set({ isUpdating: true, error: null });
      try {
        const response = await apiClient.post<AdminOrder>(
          `/api/orders/${orderId}/update-status/`,
          { status },
        );

        // Actualizar la orden en la lista y el detalle seleccionado
        set(state => ({
          orders: state.orders.map(order =>
            order.id === orderId ? { ...order, status } : order,
          ),
          selectedOrder:
            state.selectedOrder?.id === orderId
              ? response.data
              : state.selectedOrder,
          isUpdating: false,
        }));
      } catch (error: any) {
        set({
          error:
            error?.response?.data?.detail || 'Error al actualizar el estado',
          isUpdating: false,
        });
        throw error; // Re-lanzar para que el modal pueda manejarlo
      }
    },

    setStatusFilter: (filter: OrderStatus | 'all') => {
      set({ statusFilter: filter, currentPage: 1 });
      // Recargar pedidos con el nuevo filtro
      get().loadOrders(1);
    },

    resetError: () => set({ error: null }),
  }),
);
```

**Puntos clave del store:**
- `loadOrders` acepta un número de página para paginación. El backend de Django retorna `count` y `results` usando `PageNumberPagination`.
- `updateOrderStatus` actualiza la lista local de forma optimista luego de recibir la respuesta exitosa del servidor.
- `setStatusFilter` también recarga los pedidos inmediatamente con el nuevo filtro aplicado.

---

## 10.2 AdminOrdersScreen

**Archivo:** `src/presentation/screens/admin/AdminOrdersScreen.tsx`

Pantalla principal del panel admin de pedidos. Muestra todos los pedidos del sistema con pestañas de filtro por estado y soporte de paginación infinita.

```typescript
import React, { useEffect, useCallback } from 'react';
import {
  View,
  FlatList,
  TouchableOpacity,
  Text,
  StyleSheet,
  ActivityIndicator,
  RefreshControl,
} from 'react-native';
import { NativeStackScreenProps } from '@react-navigation/native-stack';
import { AdminStackParamList } from '../../../presentation/navigation/admin.navigator';
import { useAdminOrdersStore } from '../../../domain/store/admin.orders.store';
import { AdminOrder, OrderStatus } from '../../../domain/model/admin.orders.types';
import { Colors } from '../../../theme/colors';
import { AdminOrderCard } from '../components/AdminOrderCard';

type Props = NativeStackScreenProps<AdminStackParamList, 'AdminOrders'>;

const STATUS_FILTERS: Array<{ label: string; value: OrderStatus | 'all' }> = [
  { label: 'Todos', value: 'all' },
  { label: 'Pendiente', value: 'pending' },
  { label: 'Confirmado', value: 'confirmed' },
  { label: 'Enviado', value: 'shipped' },
  { label: 'Entregado', value: 'delivered' },
  { label: 'Cancelado', value: 'cancelled' },
];

export const AdminOrdersScreen: React.FC<Props> = ({ navigation }) => {
  const {
    orders,
    isLoading,
    statusFilter,
    currentPage,
    totalPages,
    loadOrders,
    setStatusFilter,
  } = useAdminOrdersStore();

  useEffect(() => {
    loadOrders(1);
  }, []);

  const handleOrderPress = useCallback(
    (order: AdminOrder) => {
      navigation.navigate('AdminOrderDetail', { orderId: order.id });
    },
    [navigation],
  );

  const handleLoadMore = useCallback(() => {
    if (!isLoading && currentPage < totalPages) {
      loadOrders(currentPage + 1);
    }
  }, [isLoading, currentPage, totalPages, loadOrders]);

  const renderFilterTab = (item: (typeof STATUS_FILTERS)[0]) => {
    const isActive = statusFilter === item.value;
    return (
      <TouchableOpacity
        key={item.value}
        style={[styles.filterTab, isActive && styles.filterTabActive]}
        onPress={() => setStatusFilter(item.value)}>
        <Text
          style={[styles.filterTabText, isActive && styles.filterTabTextActive]}>
          {item.label}
        </Text>
      </TouchableOpacity>
    );
  };

  const renderItem = useCallback(
    ({ item }: { item: AdminOrder }) => (
      <AdminOrderCard order={item} onPress={() => handleOrderPress(item)} />
    ),
    [handleOrderPress],
  );

  const renderFooter = () => {
    if (!isLoading || orders.length === 0) return null;
    return (
      <ActivityIndicator
        size="small"
        color={Colors.accent}
        style={{ marginVertical: 16 }}
      />
    );
  };

  return (
    <View style={styles.container}>
      {/* Pestañas de filtro por estado */}
      <View style={styles.filterRow}>
        <FlatList
          horizontal
          data={STATUS_FILTERS}
          keyExtractor={item => item.value}
          renderItem={({ item }) => renderFilterTab(item)}
          showsHorizontalScrollIndicator={false}
          contentContainerStyle={styles.filterList}
        />
      </View>

      {/* Lista de pedidos */}
      <FlatList
        data={orders}
        keyExtractor={item => item.id.toString()}
        renderItem={renderItem}
        contentContainerStyle={styles.listContent}
        onEndReached={handleLoadMore}
        onEndReachedThreshold={0.3}
        ListFooterComponent={renderFooter}
        refreshControl={
          <RefreshControl
            refreshing={isLoading && orders.length === 0}
            onRefresh={() => loadOrders(1)}
            tintColor={Colors.accent}
          />
        }
        ListEmptyComponent={
          !isLoading ? (
            <View style={styles.emptyContainer}>
              <Text style={styles.emptyText}>No hay pedidos con este filtro</Text>
            </View>
          ) : null
        }
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  filterRow: {
    backgroundColor: Colors.surface,
    paddingVertical: 8,
    borderBottomWidth: 1,
    borderBottomColor: Colors.border,
  },
  filterList: {
    paddingHorizontal: 12,
    gap: 8,
  },
  filterTab: {
    paddingHorizontal: 14,
    paddingVertical: 6,
    borderRadius: 20,
    borderWidth: 1,
    borderColor: Colors.border,
    backgroundColor: Colors.surface2,
  },
  filterTabActive: {
    backgroundColor: Colors.accent,
    borderColor: Colors.accent,
  },
  filterTabText: {
    color: Colors.textSecondary,
    fontSize: 13,
    fontWeight: '500',
  },
  filterTabTextActive: {
    color: Colors.textPrimary,
  },
  listContent: {
    padding: 16,
    gap: 12,
  },
  emptyContainer: {
    flex: 1,
    alignItems: 'center',
    paddingTop: 60,
  },
  emptyText: {
    color: Colors.textSecondary,
    fontSize: 16,
  },
});
```

### Componente AdminOrderCard

Crea `src/presentation/screens/admin/components/AdminOrderCard.tsx`:

```typescript
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { AdminOrder, OrderStatus } from '../../../domain/model/admin.orders.types';
import { Colors } from '../../../theme/colors';

const STATUS_COLORS: Record<OrderStatus, string> = {
  pending: '#FFA500',
  confirmed: Colors.accent,
  shipped: '#2196F3',
  delivered: Colors.success,
  cancelled: Colors.error,
};

const STATUS_LABELS: Record<OrderStatus, string> = {
  pending: 'Pendiente',
  confirmed: 'Confirmado',
  shipped: 'Enviado',
  delivered: 'Entregado',
  cancelled: 'Cancelado',
};

interface Props {
  order: AdminOrder;
  onPress: () => void;
}

export const AdminOrderCard: React.FC<Props> = ({ order, onPress }) => {
  const statusColor = STATUS_COLORS[order.status];
  const statusLabel = STATUS_LABELS[order.status];
  const date = new Date(order.created_at).toLocaleDateString('es-EC', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
  });

  return (
    <TouchableOpacity style={styles.card} onPress={onPress} activeOpacity={0.7}>
      <View style={styles.header}>
        <Text style={styles.orderId}>Pedido #{order.id}</Text>
        <View
          style={[
            styles.statusBadge,
            { backgroundColor: statusColor + '22' },
          ]}>
          <Text style={[styles.statusText, { color: statusColor }]}>
            {statusLabel}
          </Text>
        </View>
      </View>

      <View style={styles.userRow}>
        <Text style={styles.label}>Usuario:</Text>
        <Text style={styles.value}>{order.user_username}</Text>
      </View>
      <View style={styles.userRow}>
        <Text style={styles.label}>Email:</Text>
        <Text style={styles.value}>{order.user_email}</Text>
      </View>

      <View style={styles.footer}>
        <Text style={styles.date}>{date}</Text>
        <Text style={styles.total}>$ {parseFloat(order.total).toFixed(2)}</Text>
      </View>
    </TouchableOpacity>
  );
};

const styles = StyleSheet.create({
  card: {
    backgroundColor: Colors.surface,
    borderRadius: 12,
    padding: 16,
    borderWidth: 1,
    borderColor: Colors.border,
    gap: 8,
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  orderId: {
    color: Colors.textPrimary,
    fontSize: 15,
    fontWeight: '700',
  },
  statusBadge: {
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 12,
  },
  statusText: {
    fontSize: 12,
    fontWeight: '600',
  },
  userRow: {
    flexDirection: 'row',
    gap: 6,
  },
  label: {
    color: Colors.textSecondary,
    fontSize: 13,
  },
  value: {
    color: Colors.textPrimary,
    fontSize: 13,
    fontWeight: '500',
  },
  footer: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginTop: 4,
    paddingTop: 8,
    borderTopWidth: 1,
    borderTopColor: Colors.border,
  },
  date: {
    color: Colors.textSecondary,
    fontSize: 12,
  },
  total: {
    color: Colors.accent,
    fontSize: 16,
    fontWeight: '700',
  },
});
```

---

## 10.3 OrderStatusModal

**Archivo:** `src/presentation/screens/admin/components/OrderStatusModal.tsx`

Bottom sheet modal para cambiar el estado de un pedido. Muestra el estado actual y permite seleccionar uno nuevo con una UI clara que comunica qué significa cada estado.

```typescript
import React, { useState } from 'react';
import {
  Modal,
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
  Alert,
} from 'react-native';
import { OrderStatus } from '../../../domain/model/admin.orders.types';
import { Colors } from '../../../theme/colors';

const VALID_STATUSES: Array<{
  value: OrderStatus;
  label: string;
  description: string;
}> = [
  {
    value: 'pending',
    label: 'Pendiente',
    description: 'Pedido recibido, sin procesar',
  },
  {
    value: 'confirmed',
    label: 'Confirmado',
    description: 'Pago verificado, en preparación',
  },
  {
    value: 'shipped',
    label: 'Enviado',
    description: 'Pedido en camino al cliente',
  },
  {
    value: 'delivered',
    label: 'Entregado',
    description: 'El cliente recibió el pedido',
  },
  {
    value: 'cancelled',
    label: 'Cancelado',
    description: 'Pedido cancelado',
  },
];

const STATUS_COLORS: Record<OrderStatus, string> = {
  pending: '#FFA500',
  confirmed: Colors.accent,
  shipped: '#2196F3',
  delivered: Colors.success,
  cancelled: Colors.error,
};

interface Props {
  visible: boolean;
  currentStatus: OrderStatus;
  orderId: number;
  isUpdating: boolean;
  onClose: () => void;
  onConfirm: (status: OrderStatus) => Promise<void>;
}

export const OrderStatusModal: React.FC<Props> = ({
  visible,
  currentStatus,
  orderId,
  isUpdating,
  onClose,
  onConfirm,
}) => {
  const [selectedStatus, setSelectedStatus] =
    useState<OrderStatus>(currentStatus);

  const handleConfirm = async () => {
    if (selectedStatus === currentStatus) {
      onClose();
      return;
    }

    try {
      await onConfirm(selectedStatus);
      onClose();
    } catch {
      Alert.alert('Error', 'No se pudo actualizar el estado. Intenta de nuevo.');
    }
  };

  return (
    <Modal
      visible={visible}
      transparent
      animationType="slide"
      onRequestClose={onClose}>
      <TouchableOpacity
        style={styles.overlay}
        activeOpacity={1}
        onPress={onClose}
      />
      <View style={styles.sheet}>
        <View style={styles.handle} />

        <Text style={styles.title}>Cambiar estado</Text>
        <Text style={styles.subtitle}>Pedido #{orderId}</Text>

        <View style={styles.optionsList}>
          {VALID_STATUSES.map(status => {
            const isSelected = selectedStatus === status.value;
            const isCurrent = currentStatus === status.value;
            const color = STATUS_COLORS[status.value];

            return (
              <TouchableOpacity
                key={status.value}
                style={[
                  styles.option,
                  isSelected && styles.optionSelected,
                  { borderLeftColor: color, borderLeftWidth: 3 },
                ]}
                onPress={() => setSelectedStatus(status.value)}
                disabled={isUpdating}>
                <View style={styles.optionContent}>
                  <View style={styles.optionHeader}>
                    <Text
                      style={[
                        styles.optionLabel,
                        isSelected && { color },
                      ]}>
                      {status.label}
                    </Text>
                    {isCurrent && (
                      <Text style={styles.currentBadge}>Actual</Text>
                    )}
                  </View>
                  <Text style={styles.optionDescription}>
                    {status.description}
                  </Text>
                </View>
                <View
                  style={[
                    styles.radio,
                    isSelected && { borderColor: color },
                  ]}>
                  {isSelected && (
                    <View
                      style={[styles.radioDot, { backgroundColor: color }]}
                    />
                  )}
                </View>
              </TouchableOpacity>
            );
          })}
        </View>

        <View style={styles.actions}>
          <TouchableOpacity
            style={styles.cancelButton}
            onPress={onClose}
            disabled={isUpdating}>
            <Text style={styles.cancelText}>Cancelar</Text>
          </TouchableOpacity>

          <TouchableOpacity
            style={[
              styles.confirmButton,
              isUpdating && styles.confirmButtonDisabled,
            ]}
            onPress={handleConfirm}
            disabled={isUpdating}>
            {isUpdating ? (
              <ActivityIndicator size="small" color={Colors.textPrimary} />
            ) : (
              <Text style={styles.confirmText}>Confirmar</Text>
            )}
          </TouchableOpacity>
        </View>
      </View>
    </Modal>
  );
};

const styles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.5)',
  },
  sheet: {
    backgroundColor: Colors.surface,
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    paddingHorizontal: 20,
    paddingBottom: 34,
    paddingTop: 12,
  },
  handle: {
    width: 40,
    height: 4,
    backgroundColor: Colors.border,
    borderRadius: 2,
    alignSelf: 'center',
    marginBottom: 16,
  },
  title: {
    color: Colors.textPrimary,
    fontSize: 18,
    fontWeight: '700',
    marginBottom: 4,
  },
  subtitle: {
    color: Colors.textSecondary,
    fontSize: 14,
    marginBottom: 20,
  },
  optionsList: {
    gap: 10,
    marginBottom: 24,
  },
  option: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: Colors.surface2,
    borderRadius: 10,
    padding: 14,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  optionSelected: {
    borderColor: Colors.accent,
  },
  optionContent: {
    flex: 1,
    gap: 2,
  },
  optionHeader: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
  },
  optionLabel: {
    color: Colors.textPrimary,
    fontSize: 15,
    fontWeight: '600',
  },
  currentBadge: {
    color: Colors.textSecondary,
    fontSize: 11,
    fontStyle: 'italic',
  },
  optionDescription: {
    color: Colors.textSecondary,
    fontSize: 12,
  },
  radio: {
    width: 20,
    height: 20,
    borderRadius: 10,
    borderWidth: 2,
    borderColor: Colors.border,
    alignItems: 'center',
    justifyContent: 'center',
  },
  radioDot: {
    width: 10,
    height: 10,
    borderRadius: 5,
  },
  actions: {
    flexDirection: 'row',
    gap: 12,
  },
  cancelButton: {
    flex: 1,
    paddingVertical: 14,
    borderRadius: 10,
    borderWidth: 1,
    borderColor: Colors.border,
    alignItems: 'center',
  },
  cancelText: {
    color: Colors.textSecondary,
    fontSize: 15,
    fontWeight: '600',
  },
  confirmButton: {
    flex: 2,
    paddingVertical: 14,
    borderRadius: 10,
    backgroundColor: Colors.accent,
    alignItems: 'center',
  },
  confirmButtonDisabled: {
    opacity: 0.6,
  },
  confirmText: {
    color: Colors.textPrimary,
    fontSize: 15,
    fontWeight: '700',
  },
});
```

---

## 10.4 AdminOrderDetailScreen

**Archivo:** `src/presentation/screens/admin/AdminOrderDetailScreen.tsx`

Pantalla de detalle completo de un pedido para el administrador, con información del usuario, items del pedido y botón para cambiar el estado.

```typescript
import React, { useEffect, useState } from 'react';
import {
  View,
  ScrollView,
  Text,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
} from 'react-native';
import { NativeStackScreenProps } from '@react-navigation/native-stack';
import { AdminStackParamList } from '../../../presentation/navigation/admin.navigator';
import { useAdminOrdersStore } from '../../../domain/store/admin.orders.store';
import { OrderStatusModal } from './components/OrderStatusModal';
import { Colors } from '../../../theme/colors';
import { OrderStatus } from '../../../domain/model/admin.orders.types';

type Props = NativeStackScreenProps<AdminStackParamList, 'AdminOrderDetail'>;

export const AdminOrderDetailScreen: React.FC<Props> = ({ route }) => {
  const { orderId } = route.params;
  const [statusModalVisible, setStatusModalVisible] = useState(false);

  const {
    selectedOrder,
    isLoading,
    isUpdating,
    error,
    loadOrderDetail,
    updateOrderStatus,
  } = useAdminOrdersStore();

  useEffect(() => {
    loadOrderDetail(orderId);
  }, [orderId]);

  const handleUpdateStatus = async (status: OrderStatus) => {
    await updateOrderStatus(orderId, status);
  };

  if (isLoading && !selectedOrder) {
    return (
      <View style={styles.centered}>
        <ActivityIndicator size="large" color={Colors.accent} />
      </View>
    );
  }

  if (error && !selectedOrder) {
    return (
      <View style={styles.centered}>
        <Text style={styles.errorText}>{error}</Text>
      </View>
    );
  }

  if (!selectedOrder) return null;

  const date = new Date(selectedOrder.created_at).toLocaleDateString('es-EC', {
    day: '2-digit',
    month: 'long',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  });

  return (
    <>
      <ScrollView
        style={styles.container}
        contentContainerStyle={styles.content}>

        {/* Información del pedido */}
        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Información del Pedido</Text>
          <InfoRow label="Pedido #" value={selectedOrder.id.toString()} />
          <InfoRow label="Fecha" value={date} />
          <InfoRow
            label="Estado"
            value={selectedOrder.status.toUpperCase()}
            accent
          />
          <InfoRow
            label="Total"
            value={`$ ${parseFloat(selectedOrder.total).toFixed(2)}`}
            accent
          />
        </View>

        {/* Datos del cliente */}
        <View style={styles.section}>
          <Text style={styles.sectionTitle}>Datos del Cliente</Text>
          <InfoRow label="Usuario" value={selectedOrder.user_username} />
          <InfoRow label="Email" value={selectedOrder.user_email} />
          <InfoRow label="Dirección" value={selectedOrder.shipping_address} />
        </View>

        {/* Items del pedido */}
        <View style={styles.section}>
          <Text style={styles.sectionTitle}>
            Productos ({selectedOrder.items.length})
          </Text>
          {selectedOrder.items.map(item => (
            <View key={item.id} style={styles.itemRow}>
              <View style={styles.itemInfo}>
                <Text style={styles.itemName}>{item.product_name}</Text>
                <Text style={styles.itemSku}>SKU: {item.product_sku}</Text>
              </View>
              <View style={styles.itemPrices}>
                <Text style={styles.itemQty}>x{item.quantity}</Text>
                <Text style={styles.itemSubtotal}>
                  $ {parseFloat(item.subtotal).toFixed(2)}
                </Text>
              </View>
            </View>
          ))}
        </View>

        {/* Botón para cambiar estado */}
        <TouchableOpacity
          style={styles.changeStatusButton}
          onPress={() => setStatusModalVisible(true)}
          disabled={isUpdating}>
          {isUpdating ? (
            <ActivityIndicator size="small" color={Colors.textPrimary} />
          ) : (
            <Text style={styles.changeStatusText}>Cambiar Estado del Pedido</Text>
          )}
        </TouchableOpacity>
      </ScrollView>

      <OrderStatusModal
        visible={statusModalVisible}
        currentStatus={selectedOrder.status}
        orderId={selectedOrder.id}
        isUpdating={isUpdating}
        onClose={() => setStatusModalVisible(false)}
        onConfirm={handleUpdateStatus}
      />
    </>
  );
};

interface InfoRowProps {
  label: string;
  value: string;
  accent?: boolean;
}

const InfoRow: React.FC<InfoRowProps> = ({ label, value, accent }) => (
  <View style={styles.infoRow}>
    <Text style={styles.infoLabel}>{label}</Text>
    <Text style={[styles.infoValue, accent && styles.infoValueAccent]}>
      {value}
    </Text>
  </View>
);

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  content: {
    padding: 16,
    gap: 16,
    paddingBottom: 40,
  },
  centered: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: Colors.background,
  },
  errorText: {
    color: Colors.error,
    fontSize: 15,
  },
  section: {
    backgroundColor: Colors.surface,
    borderRadius: 12,
    padding: 16,
    borderWidth: 1,
    borderColor: Colors.border,
    gap: 10,
  },
  sectionTitle: {
    color: Colors.textPrimary,
    fontSize: 15,
    fontWeight: '700',
    marginBottom: 4,
    borderBottomWidth: 1,
    borderBottomColor: Colors.border,
    paddingBottom: 8,
  },
  infoRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'flex-start',
    gap: 8,
  },
  infoLabel: {
    color: Colors.textSecondary,
    fontSize: 13,
    flex: 1,
  },
  infoValue: {
    color: Colors.textPrimary,
    fontSize: 13,
    fontWeight: '500',
    flex: 2,
    textAlign: 'right',
  },
  infoValueAccent: {
    color: Colors.accent,
    fontWeight: '700',
  },
  itemRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 8,
    borderBottomWidth: 1,
    borderBottomColor: Colors.border,
  },
  itemInfo: {
    flex: 2,
    gap: 2,
  },
  itemName: {
    color: Colors.textPrimary,
    fontSize: 14,
    fontWeight: '500',
  },
  itemSku: {
    color: Colors.textSecondary,
    fontSize: 11,
  },
  itemPrices: {
    flex: 1,
    alignItems: 'flex-end',
    gap: 2,
  },
  itemQty: {
    color: Colors.textSecondary,
    fontSize: 12,
  },
  itemSubtotal: {
    color: Colors.textPrimary,
    fontSize: 14,
    fontWeight: '600',
  },
  changeStatusButton: {
    backgroundColor: Colors.accent,
    borderRadius: 12,
    paddingVertical: 16,
    alignItems: 'center',
    marginTop: 8,
  },
  changeStatusText: {
    color: Colors.textPrimary,
    fontSize: 16,
    fontWeight: '700',
  },
});
```

### Configuración de la navegación admin

Agrega las rutas al stack navigator en `src/presentation/navigation/admin.navigator.tsx`:

```typescript
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { AdminOrdersScreen } from '../screens/admin/AdminOrdersScreen';
import { AdminOrderDetailScreen } from '../screens/admin/AdminOrderDetailScreen';
import { Colors } from '../../theme/colors';

export type AdminStackParamList = {
  AdminOrders: undefined;
  AdminOrderDetail: { orderId: number };
  AdminUsers: undefined;
  // ... otras pantallas admin
};

const Stack = createNativeStackNavigator<AdminStackParamList>();

export const AdminNavigator = () => (
  <Stack.Navigator
    screenOptions={{
      headerStyle: { backgroundColor: Colors.surface },
      headerTintColor: Colors.textPrimary,
      headerTitleStyle: { fontWeight: '700' },
    }}>
    <Stack.Screen
      name="AdminOrders"
      component={AdminOrdersScreen}
      options={{ title: 'Gestión de Pedidos' }}
    />
    <Stack.Screen
      name="AdminOrderDetail"
      component={AdminOrderDetailScreen}
      options={{ title: 'Detalle de Pedido' }}
    />
  </Stack.Navigator>
);
```

---

> **Checkpoint — Módulo 10**
>
> Verifica que puedes:
> - [ ] Iniciar sesión con un usuario `is_staff = true`
> - [ ] Ver la pantalla `AdminOrdersScreen` con todos los pedidos del sistema
> - [ ] Filtrar pedidos usando las pestañas de estado (Todos, Pendiente, Confirmado, etc.)
> - [ ] Navegar al detalle de un pedido y ver los datos del cliente, items y total
> - [ ] Cambiar el estado de un pedido con `OrderStatusModal` y ver la actualización reflejada en la lista sin recargar la pantalla

---

**Siguiente módulo:** Módulo 11 — Panel Admin: Gestión de Usuarios
