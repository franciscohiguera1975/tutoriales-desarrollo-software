# ShopApp React Native — Módulo 14

## Pulido y Optimización — Loading skeletons, error boundaries y UX

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulos 1–13 completados.

---

> **Objetivo**
> Mejorar la experiencia de usuario de la app con skeletons de carga animados, manejo global de errores, estados vacíos descriptivos, notificaciones toast y optimizaciones de rendimiento en listas largas.

---

> **Checkpoint final**
> Al terminar este módulo deberás poder:
> - Ver el skeleton animado mientras se cargan los productos (en lugar de un spinner genérico)
> - Recuperarte de un error de renderizado con el botón "Reintentar" del ErrorBoundary
> - Ver mensajes de estado vacío cuando no hay resultados
> - Recibir notificaciones toast al agregar un producto al carrito
> - El scroll de la lista de productos es fluido en dispositivos de gama media

---

## 14.1 Loading skeleton para ProductCard

**Archivo:** `src/presentation/components/ProductCardSkeleton.tsx`

El skeleton usa la API `Animated` de React Native para crear un efecto shimmer (brillo que se desplaza de izquierda a derecha). Las dimensiones coinciden con las del `ProductCard` real para que la transición al contenido sea imperceptible.

```typescript
import React, { useEffect, useRef } from 'react';
import { View, Animated, StyleSheet, Easing } from 'react-native';
import { Colors } from '../../../core/theme/colors';

// Componente base del efecto shimmer reutilizable
interface ShimmerProps {
  width: number | string;
  height: number;
  borderRadius?: number;
  style?: object;
}

const Shimmer: React.FC<ShimmerProps> = ({
  width,
  height,
  borderRadius = 6,
  style,
}) => {
  const shimmerAnim = useRef(new Animated.Value(0)).current;

  useEffect(() => {
    const loop = Animated.loop(
      Animated.sequence([
        Animated.timing(shimmerAnim, {
          toValue: 1,
          duration: 900,
          easing: Easing.inOut(Easing.ease),
          useNativeDriver: false, // backgroundColor no es compatible con Native Driver
        }),
        Animated.timing(shimmerAnim, {
          toValue: 0,
          duration: 900,
          easing: Easing.inOut(Easing.ease),
          useNativeDriver: false,
        }),
      ]),
    );
    loop.start();
    return () => loop.stop();
  }, [shimmerAnim]);

  const backgroundColor = shimmerAnim.interpolate({
    inputRange: [0, 1],
    outputRange: [Colors.surface2, Colors.borderLight],
  });

  return (
    <Animated.View
      style={[
        { width, height, borderRadius, backgroundColor },
        style,
      ]}
    />
  );
};

// Skeleton de una tarjeta de producto completa
export const ProductCardSkeleton: React.FC = () => {
  return (
    <View style={styles.card}>
      {/* Imagen del producto */}
      <Shimmer width="100%" height={140} borderRadius={10} />

      <View style={styles.content}>
        {/* Categoría */}
        <Shimmer width={80} height={12} />

        {/* Nombre del producto (dos líneas) */}
        <View style={styles.nameArea}>
          <Shimmer width="90%" height={16} />
          <Shimmer width="65%" height={16} />
        </View>

        {/* Precio y botón */}
        <View style={styles.footer}>
          <Shimmer width={80} height={22} />
          <Shimmer width={36} height={36} borderRadius={18} />
        </View>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  card: {
    backgroundColor: Colors.surface,
    borderRadius: 14,
    overflow: 'hidden',
    borderWidth: 1,
    borderColor: Colors.border,
    width: '48%', // Para grid de 2 columnas
  },
  content: {
    padding: 12,
    gap: 10,
  },
  nameArea: {
    gap: 6,
  },
  footer: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginTop: 4,
  },
});
```

---

## 14.2 Loading skeleton para listas

**Archivo:** `src/core/components/SkeletonList.tsx`

Componente genérico que renderiza N copias del skeleton pasado como `renderItem` mientras `isLoading` es `true`.

```typescript
import React from 'react';
import { View, StyleSheet } from 'react-native';

interface Props {
  isLoading: boolean;
  count?: number;
  renderSkeleton: () => React.ReactElement;
  numColumns?: number;
  children: React.ReactNode;
}

export const SkeletonList: React.FC<Props> = ({
  isLoading,
  count = 6,
  renderSkeleton,
  numColumns = 1,
  children,
}) => {
  if (!isLoading) {
    return <>{children}</>;
  }

  const skeletons = Array.from({ length: count }, (_, i) => i);

  return (
    <View style={[styles.grid, numColumns > 1 && styles.multiColumn]}>
      {skeletons.map(i => (
        <React.Fragment key={i}>{renderSkeleton()}</React.Fragment>
      ))}
    </View>
  );
};

const styles = StyleSheet.create({
  grid: {
    flex: 1,
  },
  multiColumn: {
    flexDirection: 'row',
    flexWrap: 'wrap',
    gap: 12,
    padding: 16,
  },
});
```

### Uso en CatalogScreen

```typescript
// En CatalogScreen.tsx, reemplazar el indicador de carga:
import { SkeletonList } from '../../../core/components/SkeletonList';
import { ProductCardSkeleton } from '../../../presentation/components/ProductCardSkeleton';

// En el JSX:
<SkeletonList
  isLoading={isLoading && products.length === 0}
  count={6}
  renderSkeleton={() => <ProductCardSkeleton />}
  numColumns={2}>
  <FlatList
    data={products}
    numColumns={2}
    {/* ... resto de props */}
  />
</SkeletonList>
```

---

## 14.3 Error boundary

**Archivo:** `src/core/components/ErrorBoundary.tsx`

Los Error Boundaries deben implementarse como componentes de clase en React, ya que los hooks no pueden capturar errores de renderizado. Este componente envuelve secciones de la app para mostrar una pantalla amigable cuando ocurre un error inesperado.

```typescript
import React, { Component, ReactNode, ErrorInfo } from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  ScrollView,
} from 'react-native';
import { Colors } from '../theme/colors';

interface Props {
  children: ReactNode;
  fallback?: ReactNode; // UI alternativa personalizada opcional
}

interface State {
  hasError: boolean;
  error: Error | null;
  errorInfo: ErrorInfo | null;
}

export class ErrorBoundary extends Component<Props, State> {
  constructor(props: Props) {
    super(props);
    this.state = { hasError: false, error: null, errorInfo: null };
  }

  static getDerivedStateFromError(error: Error): Partial<State> {
    // Actualizar el estado para que el siguiente render muestre el fallback
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo): void {
    // Aquí se puede enviar el error a un servicio de monitoreo (Sentry, Bugsnag, etc.)
    console.error('[ErrorBoundary] Error capturado:', error);
    console.error('[ErrorBoundary] Info del componente:', errorInfo);
    this.setState({ errorInfo });
  }

  handleRetry = (): void => {
    this.setState({ hasError: false, error: null, errorInfo: null });
  };

  render(): ReactNode {
    if (this.state.hasError) {
      // Si se provee un fallback personalizado, usarlo
      if (this.props.fallback) {
        return this.props.fallback;
      }

      // UI de error predeterminada
      return (
        <View style={styles.container}>
          <Text style={styles.emoji}>⚠️</Text>
          <Text style={styles.title}>Algo salió mal</Text>
          <Text style={styles.message}>
            Ocurrió un error inesperado. Puedes intentar recargar esta sección.
          </Text>

          <TouchableOpacity style={styles.retryButton} onPress={this.handleRetry}>
            <Text style={styles.retryText}>Reintentar</Text>
          </TouchableOpacity>

          {/* Mostrar detalles técnicos solo en desarrollo */}
          {__DEV__ && this.state.error && (
            <ScrollView style={styles.debugContainer}>
              <Text style={styles.debugTitle}>Error (solo visible en desarrollo):</Text>
              <Text style={styles.debugText}>{this.state.error.toString()}</Text>
              {this.state.errorInfo && (
                <Text style={styles.debugText}>
                  {this.state.errorInfo.componentStack}
                </Text>
              )}
            </ScrollView>
          )}
        </View>
      );
    }

    return this.props.children;
  }
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: Colors.background,
    padding: 32,
    gap: 16,
  },
  emoji: {
    fontSize: 56,
  },
  title: {
    color: Colors.textPrimary,
    fontSize: 22,
    fontWeight: '700',
    textAlign: 'center',
  },
  message: {
    color: Colors.textSecondary,
    fontSize: 15,
    textAlign: 'center',
    lineHeight: 22,
    maxWidth: 300,
  },
  retryButton: {
    backgroundColor: Colors.accent,
    paddingHorizontal: 32,
    paddingVertical: 14,
    borderRadius: 12,
    marginTop: 8,
  },
  retryText: {
    color: Colors.textPrimary,
    fontSize: 16,
    fontWeight: '700',
  },
  debugContainer: {
    backgroundColor: Colors.surface,
    borderRadius: 10,
    padding: 12,
    marginTop: 16,
    maxHeight: 200,
    width: '100%',
    borderWidth: 1,
    borderColor: Colors.error + '55',
  },
  debugTitle: {
    color: Colors.error,
    fontSize: 12,
    fontWeight: '700',
    marginBottom: 6,
  },
  debugText: {
    color: Colors.textSecondary,
    fontSize: 11,
    fontFamily: 'monospace',
  },
});
```

### Uso del ErrorBoundary

Envuelve pantallas o secciones críticas en `AppNavigator` o directamente en las pantallas:

```typescript
// En AppNavigator o en el archivo raíz App.tsx:
import { ErrorBoundary } from './src/core/components/ErrorBoundary';

// Envolver el stack completo:
<ErrorBoundary>
  <NavigationContainer>
    <AppNavigator />
  </NavigationContainer>
</ErrorBoundary>

// O envolver solo una pantalla específica en su stack:
<Stack.Screen
  name="Catalog"
  children={() => (
    <ErrorBoundary>
      <CatalogScreen />
    </ErrorBoundary>
  )}
/>
```

---

## 14.4 Empty state

**Archivo:** `src/core/components/EmptyState.tsx`

Componente reutilizable para mostrar cuando una lista no tiene resultados.

```typescript
import React from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
} from 'react-native';
import { Colors } from '../theme/colors';

interface Props {
  icon?: string;       // Emoji o ícono como string
  title: string;
  message?: string;
  actionLabel?: string;
  onAction?: () => void;
}

export const EmptyState: React.FC<Props> = ({
  icon = '📭',
  title,
  message,
  actionLabel,
  onAction,
}) => {
  return (
    <View style={styles.container}>
      <Text style={styles.icon}>{icon}</Text>
      <Text style={styles.title}>{title}</Text>
      {message && <Text style={styles.message}>{message}</Text>}
      {actionLabel && onAction && (
        <TouchableOpacity style={styles.actionButton} onPress={onAction}>
          <Text style={styles.actionText}>{actionLabel}</Text>
        </TouchableOpacity>
      )}
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 40,
    gap: 12,
    minHeight: 300,
  },
  icon: {
    fontSize: 52,
    marginBottom: 8,
  },
  title: {
    color: Colors.textPrimary,
    fontSize: 20,
    fontWeight: '700',
    textAlign: 'center',
  },
  message: {
    color: Colors.textSecondary,
    fontSize: 14,
    textAlign: 'center',
    lineHeight: 20,
    maxWidth: 280,
  },
  actionButton: {
    marginTop: 8,
    backgroundColor: Colors.accent,
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 10,
  },
  actionText: {
    color: Colors.textPrimary,
    fontSize: 15,
    fontWeight: '600',
  },
});
```

### Ejemplos de uso por pantalla

```typescript
// En CatalogScreen — sin resultados de búsqueda:
<EmptyState
  icon="🔍"
  title="Sin resultados"
  message={`No encontramos productos para "${searchQuery}"`}
  actionLabel="Limpiar búsqueda"
  onAction={() => setSearchQuery('')}
/>

// En CartScreen — carrito vacío:
<EmptyState
  icon="🛒"
  title="Tu carrito está vacío"
  message="Agrega productos desde el catálogo para comenzar"
  actionLabel="Ir al catálogo"
  onAction={() => navigation.navigate('Catalog')}
/>

// En OrdersScreen — sin pedidos:
<EmptyState
  icon="📦"
  title="Sin pedidos aún"
  message="Cuando realices tu primera compra aparecerá aquí"
/>
```

---

## 14.5 Toast notifications

**Archivo:** `src/core/utils/toast.ts`

Utilidad simple para mostrar notificaciones temporales. Usa `ToastAndroid` en Android y `Alert` en iOS (React Native no tiene un API de toast nativo en iOS).

```typescript
import { Platform, ToastAndroid, Alert } from 'react-native';

type ToastDuration = 'short' | 'long';

interface ToastOptions {
  duration?: ToastDuration;
}

/**
 * Muestra una notificación tipo toast.
 * Android: usa el ToastAndroid nativo.
 * iOS: usa un Alert que se cierra automáticamente (simulado).
 */
export const showToast = (
  message: string,
  options: ToastOptions = {},
): void => {
  const { duration = 'short' } = options;

  if (Platform.OS === 'android') {
    ToastAndroid.show(
      message,
      duration === 'short' ? ToastAndroid.SHORT : ToastAndroid.LONG,
    );
  } else {
    // En iOS, simulamos el toast con un Alert que se auto-cierra
    // Para producción considera usar una librería como react-native-toast-message
    Alert.alert('', message, [{ text: 'OK', style: 'cancel' }], {
      cancelable: true,
    });
  }
};

// Helpers de conveniencia para casos comunes
export const toast = {
  success: (message: string) => showToast(message),
  error: (message: string) => showToast(message, { duration: 'long' }),
  info: (message: string) => showToast(message),
};
```

### Uso en CartStore al agregar producto

```typescript
// En cart.store.ts, dentro de addItem:
import { toast } from '../../../core/utils/toast';

addItem: (product, quantity) => {
  set(state => {
    // ... lógica existente
  });
  toast.success(`"${product.name}" agregado al carrito`);
},
```

**Nota para producción:** Para una experiencia más rica, considera `react-native-toast-message` o `react-hot-toast`. Estas librerías permiten toasts con iconos, colores personalizados y animaciones, compatibles tanto con Android como iOS sin usar `Alert`.

---

## 14.6 Optimizar FlatList

Las `FlatList` con muchos items pueden volverse lentas si los componentes se re-renderizan en cada actualización del estado padre. Aplica estas optimizaciones en `CatalogScreen`:

```typescript
import React, { useCallback, useMemo } from 'react';
import { FlatList, ListRenderItem } from 'react-native';
import { Product } from '../types/catalog.types';

// Componente del item envuelto en React.memo
// Solo se re-renderiza si sus props cambian
const MemoizedProductCard = React.memo(ProductCard, (prev, next) => {
  // Comparación personalizada: solo re-renderizar si el producto o el handler cambia
  return prev.product.id === next.product.id &&
    prev.product.stock === next.product.stock &&
    prev.product.image_url === next.product.image_url &&
    prev.onPress === next.onPress;
});

// Dentro del componente CatalogScreen:
const ITEM_HEIGHT = 240; // Altura fija del ProductCard en la lista de 2 columnas

// useCallback garantiza que renderItem no se recrea en cada render del padre
const renderItem: ListRenderItem<Product> = useCallback(
  ({ item }) => (
    <MemoizedProductCard
      product={item}
      onPress={handleProductPress}
    />
  ),
  [handleProductPress],
);

// keyExtractor con useCallback
const keyExtractor = useCallback(
  (item: Product) => item.id.toString(),
  [],
);

// getItemLayout permite a FlatList calcular posiciones sin medir cada item
// Solo funciona correctamente cuando TODOS los items tienen la misma altura
const getItemLayout = useCallback(
  (_: any, index: number) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * Math.floor(index / 2), // Para 2 columnas
    index,
  }),
  [],
);

// Lista optimizada
<FlatList
  data={products}
  numColumns={2}
  keyExtractor={keyExtractor}
  renderItem={renderItem}
  getItemLayout={getItemLayout}       // Descomenta solo si los items tienen altura fija
  removeClippedSubviews               // Desmontar items fuera de la pantalla (Android)
  maxToRenderPerBatch={10}            // Cuántos items renderizar por lote
  windowSize={10}                     // Ventana de renderizado (5 = 5 pantallas arriba+abajo)
  initialNumToRender={8}              // Items a renderizar en el primer frame
  updateCellsBatchingPeriod={50}      // Tiempo entre lotes de actualización (ms)
  contentContainerStyle={styles.listContent}
/>
```

**Cuándo usar `getItemLayout`:** Solo cuando todos los items tienen exactamente la misma altura. Si los items tienen altura variable (por ejemplo, descripciones de longitud distinta), omite `getItemLayout` para evitar posicionamiento incorrecto.

---

## 14.7 Manejo global de errores de auth

Cuando el backend responde con `401 Unauthorized` en cualquier petición (por ejemplo, cuando el token expiró y no se pudo refrescar), la app debe desloguear al usuario automáticamente y redirigirlo a la pantalla de login.

### Sistema de eventos de autenticación

Crea `src/data/api/auth.events.ts`:

```typescript
// Canal de eventos para comunicar el interceptor de axios con el navegador
// sin crear dependencia circular
type AuthEventListener = () => void;

class AuthEventEmitter {
  private listeners: AuthEventListener[] = [];

  subscribe(listener: AuthEventListener): () => void {
    this.listeners.push(listener);
    // Retornar función de desuscripción
    return () => {
      this.listeners = this.listeners.filter(l => l !== listener);
    };
  }

  emit(): void {
    this.listeners.forEach(l => l());
  }
}

export const authEvents = new AuthEventEmitter();
```

### Configurar interceptor en api.client.ts

```typescript
// En src/data/api/api.client.ts, dentro de la configuración de interceptores:
import { authEvents } from './auth.events';

apiClient.interceptors.response.use(
  response => response,
  async error => {
    const status = error?.response?.status;

    if (status === 401) {
      // Intentar refrescar el token primero
      const refreshed = await tryRefreshToken();

      if (!refreshed) {
        // Si no se puede refrescar, emitir evento de logout
        authEvents.emit();
      }
    }

    return Promise.reject(error);
  },
);
```

### Escuchar el evento en AppNavigator

```typescript
// En src/navigation/AppNavigator.tsx:
import { useEffect } from 'react';
import { authEvents } from '../data/api/auth.events';
import { useAuthStore } from '../features/auth/store/auth.store';

export const AppNavigator: React.FC = () => {
  const { logout, isAuthenticated } = useAuthStore();

  useEffect(() => {
    // Suscribirse al evento de logout forzado
    const unsubscribe = authEvents.subscribe(() => {
      logout();
      // React Navigation redirigirá automáticamente al stack de Auth
      // gracias al condicional isAuthenticated en el navigator
    });

    return unsubscribe; // Cleanup al desmontar
  }, [logout]);

  return (
    <NavigationContainer>
      {isAuthenticated ? <MainNavigator /> : <AuthNavigator />}
    </NavigationContainer>
  );
};
```

Este patrón garantiza que:
1. El interceptor de Axios detecta el 401 irrecuperable
2. Emite un evento sin acoplarse directamente al store o al navegador
3. El AppNavigator escucha el evento y ejecuta el logout
4. React Navigation redirige automáticamente al stack de autenticación

---

> **Checkpoint — Módulo 14**
>
> Verifica que puedes:
> - [ ] Ver el skeleton animado al cargar productos por primera vez (skeleton de 6 tarjetas antes del contenido real)
> - [ ] Forzar un error de render en un componente y ver la pantalla del ErrorBoundary con el botón "Reintentar"
> - [ ] Ver el EmptyState con ícono y mensaje cuando el carrito está vacío o la búsqueda no tiene resultados
> - [ ] Agregar un producto al carrito y ver el toast de confirmación
> - [ ] En Android: el scroll de la lista de productos es fluido en un emulador/dispositivo de gama media
> - [ ] Expirar el token manualmente (eliminar `access_token` de EncryptedStorage) y verificar que la app redirige a login automáticamente en la siguiente petición

---

**Siguiente módulo:** Módulo 15 — Distribución: Build de producción para Android e iOS
