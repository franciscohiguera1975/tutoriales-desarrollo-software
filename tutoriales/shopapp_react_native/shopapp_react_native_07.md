# ShopApp React Native — Módulo 7

## Detalle de Pedido

**Objetivo:** Implementar la pantalla de detalle de un pedido mostrando la lista completa de productos (con imagen, nombre, precio unitario y subtotal), el estado del pedido con un badge de color, el precio total y las marcas de tiempo de creación y actualización.

> **Checkpoint final:** El usuario puede tocar cualquier pedido en `OrdersScreen` y ver su detalle completo con imágenes de producto, estado correctamente coloreado y todos los totales.

---

## 7.1 OrderDetailScreen

**Archivo:** `src/presentation/screens/orders/OrderDetailScreen.tsx`

Pantalla principal del detalle. Recibe el `orderId` como parámetro de ruta, solicita el pedido a la API y renderiza sus ítems usando el componente `OrderItemCard`.

```typescript
// src/presentation/screens/orders/OrderDetailScreen.tsx

import React, { useEffect, useState, useCallback } from 'react';
import {
  View,
  Text,
  StyleSheet,
  ScrollView,
  ActivityIndicator,
  TouchableOpacity,
  RefreshControl,
} from 'react-native';
import { RouteProp, useRoute } from '@react-navigation/native';
import { Colors } from '../../../constants/Colors';
import { Order } from '../../../domain/model/orders.types';
import { apiClient } from '../../../data/api/apiClient';
import { RootStackParamList } from '../../../presentation/navigation/types';
import StatusBadge from '../components/StatusBadge';
import OrderItemCard from '../components/OrderItemCard';

type DetailRoute = RouteProp<RootStackParamList, 'OrderDetail'>;

function formatDateTime(iso: string): string {
  const date = new Date(iso);
  return date.toLocaleString('es-ES', {
    day: '2-digit',
    month: 'long',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  });
}

function SectionLabel({ children }: { children: string }) {
  return <Text style={detailStyles.sectionLabel}>{children}</Text>;
}

function MetaRow({ label, value }: { label: string; value: string }) {
  return (
    <View style={detailStyles.metaRow}>
      <Text style={detailStyles.metaLabel}>{label}</Text>
      <Text style={detailStyles.metaValue}>{value}</Text>
    </View>
  );
}

export default function OrderDetailScreen() {
  const route = useRoute<DetailRoute>();
  const { orderId } = route.params;

  const [order, setOrder] = useState<Order | null>(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const fetchOrder = useCallback(async () => {
    setIsLoading(true);
    setError(null);
    try {
      const response = await apiClient.get<Order>(`/api/orders/${orderId}/`);
      setOrder(response.data);
    } catch (err: any) {
      setError(
        err.response?.data?.detail ?? 'No se pudo cargar el detalle del pedido',
      );
    } finally {
      setIsLoading(false);
    }
  }, [orderId]);

  useEffect(() => {
    fetchOrder();
  }, [fetchOrder]);

  if (isLoading) {
    return (
      <View style={detailStyles.center}>
        <ActivityIndicator size="large" color={Colors.accent} />
      </View>
    );
  }

  if (error || !order) {
    return (
      <View style={detailStyles.center}>
        <Text style={detailStyles.errorText}>
          {error ?? 'Pedido no encontrado'}
        </Text>
        <TouchableOpacity
          style={detailStyles.retryBtn}
          onPress={fetchOrder}
        >
          <Text style={detailStyles.retryText}>Reintentar</Text>
        </TouchableOpacity>
      </View>
    );
  }

  const itemCount = order.items.length;

  return (
    <ScrollView
      style={detailStyles.container}
      contentContainerStyle={detailStyles.scroll}
      refreshControl={
        <RefreshControl
          refreshing={isLoading}
          onRefresh={fetchOrder}
          tintColor={Colors.accent}
        />
      }
      showsVerticalScrollIndicator={false}
    >
      {/* Cabecera del pedido */}
      <View style={detailStyles.header}>
        <View>
          <Text style={detailStyles.orderTitle}>Pedido #{order.id}</Text>
          <Text style={detailStyles.itemCountText}>
            {itemCount} {itemCount === 1 ? 'producto' : 'productos'}
          </Text>
        </View>
        <StatusBadge status={order.status} size="large" />
      </View>

      {/* Metadatos del pedido */}
      <View style={detailStyles.metaCard}>
        <SectionLabel>Información del pedido</SectionLabel>
        <MetaRow label="Creado" value={formatDateTime(order.created_at)} />
        <MetaRow label="Actualizado" value={formatDateTime(order.updated_at)} />
      </View>

      {/* Lista de ítems */}
      <SectionLabel>Productos</SectionLabel>
      {order.items.map((item) => (
        <OrderItemCard key={item.id} item={item} />
      ))}

      {/* Resumen de precios */}
      <View style={detailStyles.summaryCard}>
        <SectionLabel>Resumen</SectionLabel>
        {order.items.map((item) => (
          <View key={item.id} style={detailStyles.summaryRow}>
            <Text style={detailStyles.summaryName} numberOfLines={1}>
              {item.product.name} × {item.quantity}
            </Text>
            <Text style={detailStyles.summarySubtotal}>
              ${parseFloat(item.subtotal).toFixed(2)}
            </Text>
          </View>
        ))}
        <View style={detailStyles.divider} />
        <View style={detailStyles.totalRow}>
          <Text style={detailStyles.totalLabel}>Total del pedido</Text>
          <Text style={detailStyles.totalValue}>
            ${parseFloat(order.total).toFixed(2)}
          </Text>
        </View>
      </View>
    </ScrollView>
  );
}

const detailStyles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  scroll: {
    padding: 16,
    paddingBottom: 40,
  },
  center: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: Colors.background,
    padding: 24,
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'flex-start',
    marginBottom: 16,
  },
  orderTitle: {
    fontSize: 22,
    fontWeight: '800',
    color: Colors.textPrimary,
    marginBottom: 4,
  },
  itemCountText: {
    fontSize: 13,
    color: Colors.textSecondary,
  },
  metaCard: {
    backgroundColor: Colors.surface,
    borderRadius: 14,
    padding: 16,
    marginBottom: 20,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  sectionLabel: {
    fontSize: 12,
    fontWeight: '700',
    color: Colors.textSecondary,
    textTransform: 'uppercase',
    letterSpacing: 1,
    marginBottom: 12,
    marginTop: 4,
  },
  metaRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    paddingVertical: 5,
  },
  metaLabel: {
    fontSize: 13,
    color: Colors.textSecondary,
  },
  metaValue: {
    fontSize: 13,
    fontWeight: '500',
    color: Colors.textPrimary,
    flexShrink: 1,
    marginLeft: 12,
    textAlign: 'right',
  },
  summaryCard: {
    backgroundColor: Colors.surface,
    borderRadius: 14,
    padding: 16,
    marginTop: 8,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  summaryRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    paddingVertical: 5,
  },
  summaryName: {
    fontSize: 13,
    color: Colors.textSecondary,
    flex: 1,
    marginRight: 12,
  },
  summarySubtotal: {
    fontSize: 13,
    fontWeight: '600',
    color: Colors.textPrimary,
  },
  divider: {
    height: 1,
    backgroundColor: Colors.border,
    marginVertical: 12,
  },
  totalRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
  },
  totalLabel: {
    fontSize: 16,
    fontWeight: '600',
    color: Colors.textPrimary,
  },
  totalValue: {
    fontSize: 20,
    fontWeight: '800',
    color: Colors.accent,
  },
  errorText: {
    fontSize: 15,
    color: Colors.error,
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
});
```

**Puntos clave:**
- La pantalla hace su propia petición a `GET /api/orders/{id}/` en lugar de leer del store, para garantizar datos siempre frescos al entrar al detalle.
- El pull-to-refresh vuelve a llamar a `fetchOrder` para actualizar el estado si cambió desde que se cargó la lista.
- `parseFloat(order.total).toFixed(2)` formatea correctamente los decimales devueltos por DRF.

---

## 7.2 StatusBadge

**Archivo:** `src/presentation/screens/orders/components/StatusBadge.tsx`

Componente reutilizable que renderiza un badge de colores según el estado del pedido. Soporta dos tamaños: `normal` (para listas) y `large` (para pantalla de detalle).

```typescript
// src/presentation/screens/orders/components/StatusBadge.tsx

import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { Colors } from '../../../../constants/Colors';
import { Order } from '../../../../domain/model/orders.types';

interface StatusBadgeProps {
  status: Order['status'] | string;
  size?: 'normal' | 'large';
}

interface StatusConfig {
  label: string;
  backgroundColor: string;
  textColor: string;
}

const STATUS_MAP: Record<string, StatusConfig> = {
  pending: {
    label: 'Pendiente',
    backgroundColor: '#FFA000',
    textColor: '#FFFFFF',
  },
  confirmed: {
    label: 'Confirmado',
    backgroundColor: '#1976D2',
    textColor: '#FFFFFF',
  },
  shipped: {
    label: 'Enviado',
    backgroundColor: '#7B1FA2',
    textColor: '#FFFFFF',
  },
  delivered: {
    label: 'Entregado',
    backgroundColor: Colors.success,
    textColor: '#FFFFFF',
  },
  cancelled: {
    label: 'Cancelado',
    backgroundColor: Colors.error,
    textColor: '#FFFFFF',
  },
};

const FALLBACK_CONFIG: StatusConfig = {
  label: 'Desconocido',
  backgroundColor: Colors.surface2,
  textColor: Colors.textSecondary,
};

export default function StatusBadge({
  status,
  size = 'normal',
}: StatusBadgeProps) {
  const config = STATUS_MAP[status] ?? {
    ...FALLBACK_CONFIG,
    label: status,
  };

  const isLarge = size === 'large';

  return (
    <View
      style={[
        styles.badge,
        { backgroundColor: config.backgroundColor },
        isLarge && styles.badgeLarge,
      ]}
    >
      <Text
        style={[
          styles.text,
          { color: config.textColor },
          isLarge && styles.textLarge,
        ]}
      >
        {config.label}
      </Text>
    </View>
  );
}

const styles = StyleSheet.create({
  badge: {
    borderRadius: 8,
    paddingHorizontal: 10,
    paddingVertical: 3,
    alignSelf: 'flex-start',
  },
  badgeLarge: {
    borderRadius: 10,
    paddingHorizontal: 14,
    paddingVertical: 6,
  },
  text: {
    fontSize: 11,
    fontWeight: '700',
    textTransform: 'uppercase',
    letterSpacing: 0.5,
  },
  textLarge: {
    fontSize: 13,
    letterSpacing: 0.8,
  },
});
```

**Tabla de colores por estado:**

| Estado | Etiqueta | Color de fondo |
|--------|----------|----------------|
| `pending` | Pendiente | `#FFA000` (amarillo ámbar) |
| `confirmed` | Confirmado | `#1976D2` (azul) |
| `shipped` | Enviado | `#7B1FA2` (púrpura) |
| `delivered` | Entregado | `#4CAF50` (verde) |
| `cancelled` | Cancelado | `#FF5252` (rojo) |

> El componente acepta cualquier `string` como `status` para ser resiliente ante nuevos estados añadidos en el backend. En ese caso muestra el texto tal cual con fondo gris.

---

## 7.3 OrderItemCard

**Archivo:** `src/presentation/screens/orders/components/OrderItemCard.tsx`

Tarjeta que representa un ítem dentro de un pedido: imagen del producto (con soporte para `FastImage`), nombre, precio unitario y cálculo del subtotal.

```typescript
// src/presentation/screens/orders/components/OrderItemCard.tsx

import React from 'react';
import { View, Text, StyleSheet, Image } from 'react-native';
import { Colors } from '../../../../constants/Colors';
import { OrderItem } from '../../../../domain/model/orders.types';

// Si tienes react-native-fast-image instalado, descomenta:
// import FastImage from 'react-native-fast-image';

interface OrderItemCardProps {
  item: OrderItem;
}

function ProductImage({ uri }: { uri: string | null }) {
  if (!uri) {
    return (
      <View style={cardStyles.imagePlaceholder}>
        <Text style={cardStyles.placeholderText}>SIN{'\n'}IMG</Text>
      </View>
    );
  }

  // Con FastImage (recomendado para producción):
  // return (
  //   <FastImage
  //     source={{
  //       uri,
  //       priority: FastImage.priority.normal,
  //       cache: FastImage.cacheControl.immutable,
  //     }}
  //     style={cardStyles.image}
  //     resizeMode={FastImage.resizeMode.cover}
  //   />
  // );

  return (
    <Image
      source={{ uri }}
      style={cardStyles.image}
      resizeMode="cover"
    />
  );
}

export default function OrderItemCard({ item }: OrderItemCardProps) {
  const unitPrice = parseFloat(item.unit_price);
  const subtotal = parseFloat(item.subtotal);

  return (
    <View style={cardStyles.card}>
      {/* Imagen del producto */}
      <ProductImage uri={item.product.image_url} />

      {/* Información principal */}
      <View style={cardStyles.info}>
        <Text style={cardStyles.productName} numberOfLines={2}>
          {item.product.name}
        </Text>

        {/* Precio unitario × cantidad */}
        <View style={cardStyles.priceRow}>
          <Text style={cardStyles.unitPrice}>${unitPrice.toFixed(2)}</Text>
          <Text style={cardStyles.separator}> × </Text>
          <Text style={cardStyles.quantity}>{item.quantity}</Text>
        </View>
      </View>

      {/* Subtotal alineado a la derecha */}
      <View style={cardStyles.subtotalContainer}>
        <Text style={cardStyles.subtotalLabel}>Subtotal</Text>
        <Text style={cardStyles.subtotal}>${subtotal.toFixed(2)}</Text>
      </View>
    </View>
  );
}

const cardStyles = StyleSheet.create({
  card: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: Colors.surface,
    borderRadius: 14,
    padding: 12,
    marginBottom: 10,
    borderWidth: 1,
    borderColor: Colors.border,
    gap: 12,
  },
  image: {
    width: 72,
    height: 72,
    borderRadius: 10,
    backgroundColor: Colors.surface2,
  },
  imagePlaceholder: {
    width: 72,
    height: 72,
    borderRadius: 10,
    backgroundColor: Colors.surface2,
    alignItems: 'center',
    justifyContent: 'center',
  },
  placeholderText: {
    fontSize: 10,
    fontWeight: '700',
    color: Colors.textSecondary,
    textAlign: 'center',
    letterSpacing: 0.5,
  },
  info: {
    flex: 1,
    justifyContent: 'center',
  },
  productName: {
    fontSize: 14,
    fontWeight: '600',
    color: Colors.textPrimary,
    marginBottom: 6,
    lineHeight: 20,
  },
  priceRow: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  unitPrice: {
    fontSize: 13,
    color: Colors.textSecondary,
  },
  separator: {
    fontSize: 12,
    color: Colors.textSecondary,
  },
  quantity: {
    fontSize: 13,
    fontWeight: '700',
    color: Colors.textSecondary,
  },
  subtotalContainer: {
    alignItems: 'flex-end',
    minWidth: 70,
  },
  subtotalLabel: {
    fontSize: 10,
    color: Colors.textSecondary,
    marginBottom: 2,
    textTransform: 'uppercase',
    letterSpacing: 0.5,
  },
  subtotal: {
    fontSize: 15,
    fontWeight: '800',
    color: Colors.accent,
  },
});
```

**Instalar react-native-fast-image (recomendado para producción):**

```bash
npm install react-native-fast-image

# iOS
cd ios && pod install && cd ..
```

`FastImage` almacena las imágenes en caché de forma agresiva, mejorando notablemente el rendimiento al hacer scroll por listas de ítems.

---

## 7.4 Navegación — Conectar OrdersScreen con OrderDetailScreen

**Archivo:** `src/presentation/screens/orders/OrdersScreen.tsx` _(verificar integración)_

En el módulo 6 ya añadimos el handler `handlePressOrder`. Este paso verifica que la ruta `OrderDetail` esté correctamente tipada y que el header muestre el ID del pedido.

### Verificar tipos de navegación

```typescript
// src/presentation/navigation/types.ts — asegúrate de que exista:

export type RootStackParamList = {
  MainTabs: { screen?: string } | undefined;
  Checkout: undefined;
  OrderSuccess: { orderId: number; total: string };
  OrderDetail: { orderId: number };   // requerido por este módulo
  // ... demás rutas
};
```

### Configurar el título dinámico en el navigator

```typescript
// src/presentation/navigation/MainNavigator.tsx — Stack.Screen para OrderDetail

<Stack.Screen
  name="OrderDetail"
  component={OrderDetailScreen}
  options={({ route }) => ({
    title: `Pedido #${(route.params as { orderId: number }).orderId}`,
    headerBackTitle: 'Pedidos',
  })}
/>
```

Esto muestra `"Pedido #42"` en el header en lugar de un título genérico.

### Handler en OrdersScreen (del módulo 6)

```typescript
// En src/presentation/screens/orders/OrdersScreen.tsx — verificar que este bloque ya está presente:

const handlePressOrder = (order: Order) => {
  navigation.navigate('OrderDetail', { orderId: order.id });
};
```

### Flujo completo de navegación de pedidos

```
MainTabs
  └── Tab: Pedidos
        └── OrdersScreen  (lista paginada con StatusBadge inline)
              └── [tap en tarjeta]
                    └── OrderDetailScreen  (orderId)
                          ├── StatusBadge  (size="large")
                          └── OrderItemCard × N
                                └── ProductImage (FastImage / Image)
```

---

## ✅ Checkpoint — Módulo 7

Verifica que el detalle de pedido funcione correctamente:

- [ ] Al tocar un pedido en `OrdersScreen`, navega a `OrderDetailScreen` y el header del stack muestra `"Pedido #ID"`.
- [ ] La pantalla de detalle carga los datos desde `GET /api/orders/{id}/` y muestra un `ActivityIndicator` mientras carga.
- [ ] Si la carga falla, se muestra el mensaje de error y el botón "Reintentar".
- [ ] `StatusBadge` con `size="large"` muestra el estado en el color correcto: amarillo (`pending`), azul (`confirmed`), púrpura (`shipped`), verde (`delivered`), rojo (`cancelled`).
- [ ] Cada `OrderItemCard` muestra la imagen del producto (o placeholder con texto `SIN IMG` si `image_url` es `null`), el nombre, el precio unitario × cantidad y el subtotal calculado.
- [ ] El bloque de resumen inferior lista todos los ítems con sus subtotales y el total general del pedido.
- [ ] Las fechas de creación y actualización se muestran en formato legible en español (`"28 de mayo de 2026, 14:30"`).
- [ ] El pull-to-refresh recarga el detalle del pedido correctamente.

---

**Siguiente módulo:** El módulo 8 implementa la gestión del perfil de usuario: visualizar datos, editar nombre e email, cambiar contraseña y cerrar sesión.
