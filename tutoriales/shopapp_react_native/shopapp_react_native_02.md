# ShopApp React Native — Módulo 2
## Axios + Interceptores JWT + EncryptedStorage + Repositorios base

> **Objetivo:** Capa de datos completa: cliente HTTP con renovación automática de tokens, EncryptedStorage para persistir la sesión y APIs listas para ser consumidas por los stores de Zustand.
> **Checkpoint final:** la pantalla de verificación llama a `GET /api/categories/` y muestra las categorías reales del backend de Django.

---

## 2.1 EncryptedStorage wrapper — `src/data/storage/secure.storage.ts`

```typescript
// src/data/storage/secure.storage.ts
import EncryptedStorage from 'react-native-encrypted-storage';

const KEYS = {
  ACCESS:   'shopapp:access',
  REFRESH:  'shopapp:refresh',
  USER_ID:  'shopapp:user_id',
  USERNAME: 'shopapp:username',
  EMAIL:    'shopapp:email',
  IS_STAFF: 'shopapp:is_staff',
} as const;

// ── Tokens ───────────────────────────────────────────────────
export async function getAccessToken(): Promise<string | null> {
  return EncryptedStorage.getItem(KEYS.ACCESS);
}

export async function getRefreshToken(): Promise<string | null> {
  return EncryptedStorage.getItem(KEYS.REFRESH);
}

export async function saveTokens(access: string, refresh: string): Promise<void> {
  await Promise.all([
    EncryptedStorage.setItem(KEYS.ACCESS,  access),
    EncryptedStorage.setItem(KEYS.REFRESH, refresh),
  ]);
}

export async function saveAccessToken(access: string): Promise<void> {
  await EncryptedStorage.setItem(KEYS.ACCESS, access);
}

// ── Usuario ──────────────────────────────────────────────────
export async function saveUser(params: {
  id:       number;
  username: string;
  email:    string;
  isStaff:  boolean;
}): Promise<void> {
  await Promise.all([
    EncryptedStorage.setItem(KEYS.USER_ID,  String(params.id)),
    EncryptedStorage.setItem(KEYS.USERNAME, params.username),
    EncryptedStorage.setItem(KEYS.EMAIL,    params.email),
    EncryptedStorage.setItem(KEYS.IS_STAFF, String(params.isStaff)),
  ]);
}

export interface StoredUser {
  id:       number;
  username: string;
  email:    string;
  isStaff:  boolean;
}

export async function getStoredUser(): Promise<StoredUser | null> {
  const [id, username, email, isStaff] = await Promise.all([
    EncryptedStorage.getItem(KEYS.USER_ID),
    EncryptedStorage.getItem(KEYS.USERNAME),
    EncryptedStorage.getItem(KEYS.EMAIL),
    EncryptedStorage.getItem(KEYS.IS_STAFF),
  ]);
  if (!id || !username) return null;
  return {
    id:       parseInt(id, 10),
    username,
    email:    email    ?? '',
    isStaff:  isStaff === 'true',
  };
}

export async function isLoggedIn(): Promise<boolean> {
  const token = await getAccessToken();
  return token !== null && token.length > 0;
}

export async function clearSession(): Promise<void> {
  await Promise.all(
    Object.values(KEYS).map(k => EncryptedStorage.removeItem(k)),
  );
}
```

---

## 2.2 Manejo de errores API — `src/data/api/api.error.ts`

```typescript
// src/data/api/api.error.ts
import {AxiosError} from 'axios';

export class ApiError extends Error {
  statusCode?: number;
  fieldErrors?: Record<string, string>;

  constructor(
    message: string,
    statusCode?: number,
    fieldErrors?: Record<string, string>,
  ) {
    super(message);
    this.name       = 'ApiError';
    this.statusCode = statusCode;
    this.fieldErrors= fieldErrors;
  }

  fieldError(field: string): string | undefined {
    return this.fieldErrors?.[field];
  }
}

export function parseApiError(error: unknown): ApiError {
  if (error instanceof ApiError) return error;

  if (error instanceof AxiosError) {
    const code   = error.response?.status;
    const data   = error.response?.data;

    if (!data) {
      return new ApiError(error.message || 'Error de conexión', code);
    }

    if (typeof data === 'object') {
      // Error tipo Django: { detail: 'mensaje' }
      if ('detail' in data) {
        return new ApiError(String(data.detail), code);
      }

      // Error tipo: { error: 'mensaje' }
      if ('error' in data) {
        return new ApiError(String(data.error), code);
      }

      // Non field errors: { non_field_errors: ['msg'] }
      if ('non_field_errors' in data) {
        const msg = Array.isArray(data.non_field_errors)
          ? data.non_field_errors[0]
          : data.non_field_errors;
        return new ApiError(String(msg), code);
      }

      // Errores por campo: { campo: ['mensaje'] }
      const fieldErrors: Record<string, string> = {};
      let firstMessage: string | undefined;
      Object.entries(data).forEach(([key, value]) => {
        const msg = Array.isArray(value) ? value[0] : String(value);
        fieldErrors[key] = msg;
        if (!firstMessage) firstMessage = `${key}: ${msg}`;
      });
      return new ApiError(
        firstMessage ?? 'Error de validación',
        code,
        fieldErrors,
      );
    }

    return new ApiError(String(data), code);
  }

  if (error instanceof Error) {
    return new ApiError(error.message);
  }

  return new ApiError('Error inesperado');
}
```

---

## 2.3 Cliente Axios con interceptores JWT — `src/data/api/axios.client.ts`

```typescript
// src/data/api/axios.client.ts
import axios, {
  AxiosError,
  AxiosInstance,
  InternalAxiosRequestConfig,
} from 'axios';
import {API_BASE_URL} from '../../core/config/api.config';
import {
  clearSession,
  getAccessToken,
  getRefreshToken,
  saveAccessToken,
  saveTokens,
} from '../storage/secure.storage';

// ── Event bus para 401 no recuperable ────────────────────────
type LogoutListener = () => void;
const logoutListeners = new Set<LogoutListener>();

export const authEvents = {
  onLogout: (fn: LogoutListener) => {
    logoutListeners.add(fn);
    return () => logoutListeners.delete(fn);
  },
  emitLogout: () => logoutListeners.forEach(fn => fn()),
};

// ── Control de renovación en curso ────────────────────────────
let isRefreshing = false;
let waitQueue: Array<{
  resolve: (token: string) => void;
  reject:  (err: unknown) => void;
}> = [];

async function refreshAccessToken(): Promise<string> {
  const refresh = await getRefreshToken();
  if (!refresh) throw new ApiError('Sin refresh token');

  const response = await axios.post(
    `${API_BASE_URL}/auth/token/refresh/`,
    {refresh},
    {headers: {'Content-Type': 'application/json'}},
  );

  const {access, refresh: newRefresh} = response.data;
  if (newRefresh) {
    await saveTokens(access, newRefresh);
  } else {
    await saveAccessToken(access);
  }
  return access as string;
}

// ── Instancia principal ────────────────────────────────────────
export const apiClient: AxiosInstance = axios.create({
  baseURL: `${API_BASE_URL}/`,
  timeout: 30_000,
  headers: {'Content-Type': 'application/json'},
});

// Interceptor REQUEST — adjuntar access token
apiClient.interceptors.request.use(
  async (config: InternalAxiosRequestConfig) => {
    const token = await getAccessToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  error => Promise.reject(error),
);

// Interceptor RESPONSE — renovar token en 401
apiClient.interceptors.response.use(
  response => response,
  async (error: AxiosError) => {
    const original = error.config as InternalAxiosRequestConfig & {_retry?: boolean};
    const status   = error.response?.status;

    const is401           = status === 401;
    const alreadyRetried  = original?._retry === true;
    const isRefreshUrl    = original?.url?.includes('/auth/token/refresh/');

    if (!is401 || alreadyRetried || isRefreshUrl) {
      return Promise.reject(error);
    }

    // Cola de peticiones mientras se renueva
    if (isRefreshing) {
      return new Promise((resolve, reject) => {
        waitQueue.push({
          resolve: (newToken: string) => {
            original._retry = true;
            original.headers.Authorization = `Bearer ${newToken}`;
            resolve(apiClient(original));
          },
          reject,
        });
      });
    }

    original._retry = true;
    isRefreshing    = true;

    try {
      const newToken = await refreshAccessToken();
      waitQueue.forEach(p => p.resolve(newToken));
      waitQueue = [];
      original.headers.Authorization = `Bearer ${newToken}`;
      return apiClient(original);
    } catch (refreshError) {
      waitQueue.forEach(p => p.reject(refreshError));
      waitQueue = [];
      await clearSession();
      authEvents.emitLogout();
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  },
);

// Importar ApiError aquí para usarlo en el refreshAccessToken
import {ApiError} from './api.error';
```

---

## 2.4 API de categorías — `src/data/api/catalog.api.ts`

```typescript
// src/data/api/catalog.api.ts
import {apiClient} from './axios.client';
import {parseApiError} from './api.error';
import {
  Category,
  PaginatedProducts,
  Product,
  ProductFilters,
} from '../../domain/model/catalog.types';

// ── Categorías ────────────────────────────────────────────────
export async function getCategories(): Promise<Category[]> {
  try {
    const {data} = await apiClient.get<Category[]>('/categories/');
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function getCategory(id: number): Promise<Category> {
  try {
    const {data} = await apiClient.get<Category>(`/categories/${id}/`);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function createCategory(payload: Partial<Category>): Promise<Category> {
  try {
    const {data} = await apiClient.post<Category>('/categories/', payload);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function updateCategory(
  id: number,
  payload: Partial<Category>,
): Promise<Category> {
  try {
    const {data} = await apiClient.patch<Category>(`/categories/${id}/`, payload);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function deleteCategory(id: number): Promise<void> {
  try {
    await apiClient.delete(`/categories/${id}/`);
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function getCategoryStats(): Promise<Record<string, number>> {
  try {
    const {data} = await apiClient.get('/categories/stats/');
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

// ── Productos ─────────────────────────────────────────────────
export async function getProducts(
  filters: ProductFilters = {},
): Promise<PaginatedProducts> {
  try {
    const params: Record<string, string | number | boolean> = {};
    if (filters.search)             params.search    = filters.search;
    if (filters.category)           params.category  = filters.category;
    if (filters.price_min != null)  params.price_min = filters.price_min;
    if (filters.price_max != null)  params.price_max = filters.price_max;
    if (filters.ordering)           params.ordering  = filters.ordering;
    if (filters.is_active != null)  params.is_active = filters.is_active;
    params.page      = filters.page      ?? 1;
    params.page_size = filters.page_size ?? 12;

    const {data} = await apiClient.get<PaginatedProducts>('/products/', {params});
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function getProduct(id: number): Promise<Product> {
  try {
    const {data} = await apiClient.get<Product>(`/products/${id}/`);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function createProduct(
  payload: Record<string, unknown>,
): Promise<Product> {
  try {
    const {data} = await apiClient.post<Product>('/products/', payload);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function updateProduct(
  id: number,
  payload: Record<string, unknown>,
): Promise<Product> {
  try {
    const {data} = await apiClient.patch<Product>(`/products/${id}/`, payload);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function deleteProduct(id: number): Promise<void> {
  try {
    await apiClient.delete(`/products/${id}/`);
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function restockProduct(
  id: number,
  quantity: number,
): Promise<{id: number; name: string; new_stock: number}> {
  try {
    const {data} = await apiClient.post(`/products/${id}/restock/`, {quantity});
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function getProductStats(): Promise<Record<string, unknown>> {
  try {
    const {data} = await apiClient.get('/products/stats/');
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}
```

---

## 2.5 API de pedidos — `src/data/api/orders.api.ts`

```typescript
// src/data/api/orders.api.ts
import {apiClient} from './axios.client';
import {parseApiError} from './api.error';
import {Order, PaginatedOrders} from '../../domain/model/orders.types';

export async function getOrders(params?: {
  page?:   number;
  status?: string;
}): Promise<PaginatedOrders> {
  try {
    const {data} = await apiClient.get<PaginatedOrders>('/orders/', {params});
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function getOrder(id: number): Promise<Order> {
  try {
    const {data} = await apiClient.get<Order>(`/orders/${id}/`);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function createOrder(): Promise<Order> {
  try {
    const {data} = await apiClient.post<Order>('/orders/', {});
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function addOrderItem(
  orderId:   number,
  productId: number,
  quantity:  number,
): Promise<Order> {
  try {
    const {data} = await apiClient.post<Order>(`/orders/${orderId}/add-item/`, {
      product_id: productId,
      quantity,
    });
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function confirmOrder(orderId: number): Promise<Order> {
  try {
    const {data} = await apiClient.post<Order>(`/orders/${orderId}/confirm/`);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function updateOrderStatus(
  orderId: number,
  status:  string,
): Promise<Order> {
  try {
    const {data} = await apiClient.post<Order>(
      `/orders/${orderId}/update-status/`,
      {status},
    );
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function getOrderStats(): Promise<Record<string, unknown>> {
  try {
    const {data} = await apiClient.get('/orders/stats/');
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}
```

---

## 2.6 API de usuarios — `src/data/api/users.api.ts`

```typescript
// src/data/api/users.api.ts
import {apiClient} from './axios.client';
import {parseApiError} from './api.error';
import {PaginatedUsers, User, UserPayload} from '../../domain/model/user.types';

export async function getUsers(params?: {
  search?:    string;
  is_staff?:  boolean;
  is_active?: boolean;
  page?:      number;
}): Promise<PaginatedUsers> {
  try {
    const {data} = await apiClient.get<PaginatedUsers>('/users/', {params});
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function createUser(payload: UserPayload): Promise<User> {
  try {
    const {data} = await apiClient.post<User>('/users/', payload);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function updateUser(
  id:      number,
  payload: Partial<UserPayload>,
): Promise<User> {
  try {
    const {data} = await apiClient.patch<User>(`/users/${id}/`, payload);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function deleteUser(id: number): Promise<void> {
  try {
    await apiClient.delete(`/users/${id}/`);
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function toggleUserActive(id: number): Promise<{is_active: boolean}> {
  try {
    const {data} = await apiClient.post(`/users/${id}/toggle-active/`);
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function getUserStats(): Promise<Record<string, number>> {
  try {
    const {data} = await apiClient.get('/users/stats/');
    return data;
  } catch (e) {
    throw parseApiError(e);
  }
}
```

---

## 2.7 API de auth — `src/data/api/auth.api.ts`

```typescript
// src/data/api/auth.api.ts
import {apiClient} from './axios.client';
import {parseApiError} from './api.error';
import {
  clearSession,
  getRefreshToken,
  saveTokens,
  saveUser,
} from '../storage/secure.storage';
import {AuthResponse, LoggedUser} from '../../domain/model/auth.types';

async function persistSession(data: AuthResponse): Promise<LoggedUser> {
  await saveTokens(data.access, data.refresh);
  await saveUser({
    id:       data.user_id,
    username: data.username,
    email:    data.email,
    isStaff:  data.is_staff,
  });
  return {
    id:       data.user_id,
    username: data.username,
    email:    data.email,
    isStaff:  data.is_staff,
  };
}

export async function login(
  username: string,
  password: string,
): Promise<LoggedUser> {
  try {
    const {data} = await apiClient.post<AuthResponse>('/auth/login/', {
      username,
      password,
    });
    return persistSession(data);
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function register(
  username:  string,
  email:     string,
  password:  string,
  password2: string,
): Promise<LoggedUser> {
  try {
    const {data} = await apiClient.post<AuthResponse>('/auth/register/', {
      username,
      email,
      password,
      password2,
    });
    return persistSession(data);
  } catch (e) {
    throw parseApiError(e);
  }
}

export async function logout(): Promise<void> {
  try {
    const refresh = await getRefreshToken();
    if (refresh) {
      await apiClient.post('/auth/logout/', {refresh});
    }
  } catch {
    // El servidor puede rechazarlo — limpiar localmente de igual forma
  } finally {
    await clearSession();
  }
}
```

---

## 2.8 Verificar conexión — actualizar `App.tsx`

```tsx
// App.tsx — agregar verificación real del backend
import React, {useEffect, useState} from 'react';
import {
  ActivityIndicator,
  SafeAreaView,
  ScrollView,
  Text,
  View,
} from 'react-native';
import {Colors} from './src/theme/colors';
import {API_BASE_URL} from './src/core/config/api.config';
import {getCategories} from './src/data/api/catalog.api';
import {Category} from './src/domain/model/catalog.types';
import './global.css';

export default function App() {
  const [categories, setCategories] = useState<Category[]>([]);
  const [status, setStatus]         = useState<'loading' | 'ok' | 'error'>('loading');
  const [errorMsg, setErrorMsg]     = useState('');

  useEffect(() => {
    getCategories()
      .then(cats => {
        setCategories(cats);
        setStatus('ok');
      })
      .catch(err => {
        setErrorMsg(err.message ?? 'Error de conexión');
        setStatus('error');
      });
  }, []);

  return (
    <SafeAreaView style={{flex: 1, backgroundColor: Colors.background}}>
      <ScrollView contentContainerStyle={{padding: 24, paddingBottom: 48}}>

        {/* Logo */}
        <View style={{alignItems: 'center', marginTop: 32, marginBottom: 32}}>
          <Text style={{
            fontSize: 40, fontWeight: '800',
            color: Colors.accent, letterSpacing: -1,
          }}>
            ShopApp
          </Text>
          <Text style={{color: Colors.textSecondary, marginTop: 6, fontSize: 14}}>
            Módulo 2 · Conexión real con Django
          </Text>
        </View>

        {/* Estado de conexión */}
        <View style={{
          backgroundColor: Colors.surface,
          borderRadius: 16,
          borderWidth: 1,
          borderColor: Colors.border,
          padding: 20,
          marginBottom: 24,
        }}>
          <Text style={{
            color: Colors.textSecondary, fontSize: 11,
            fontWeight: '700', letterSpacing: 1.2, marginBottom: 16,
          }}>
            CONEXIÓN CON EL BACKEND
          </Text>

          <View style={{
            flexDirection: 'row',
            justifyContent: 'space-between',
            marginBottom: 12,
          }}>
            <Text style={{color: Colors.textSecondary, fontSize: 13}}>API URL</Text>
            <Text style={{
              color: Colors.accent, fontSize: 11,
              fontFamily: 'monospace', maxWidth: '55%', textAlign: 'right',
            }}>
              {API_BASE_URL}
            </Text>
          </View>

          <View style={{
            backgroundColor: Colors.surface2,
            borderRadius: 10,
            padding: 16,
            alignItems: 'center',
          }}>
            {status === 'loading' && (
              <View style={{flexDirection: 'row', alignItems: 'center', gap: 10}}>
                <ActivityIndicator color={Colors.accent} size="small" />
                <Text style={{color: Colors.textSecondary, fontSize: 13}}>
                  Conectando con Django...
                </Text>
              </View>
            )}
            {status === 'ok' && (
              <Text style={{color: Colors.success, fontWeight: '700', fontSize: 14}}>
                ✅ {categories.length} categorías del backend
              </Text>
            )}
            {status === 'error' && (
              <Text style={{color: Colors.error, fontSize: 13, textAlign: 'center'}}>
                ❌ {errorMsg}
              </Text>
            )}
          </View>

          {/* Categorías */}
          {categories.slice(0, 3).map((cat, i) => (
            <View key={cat.id}>
              <View style={{
                flexDirection: 'row',
                justifyContent: 'space-between',
                paddingVertical: 10,
              }}>
                <Text style={{color: Colors.textPrimary, fontSize: 13, fontWeight: '600'}}>
                  {cat.name}
                </Text>
                <Text style={{color: Colors.accent, fontSize: 12, fontWeight: '700'}}>
                  {cat.total_products} prod.
                </Text>
              </View>
              {i < Math.min(categories.length, 3) - 1 && (
                <View style={{height: 0.5, backgroundColor: Colors.borderLight}} />
              )}
            </View>
          ))}
        </View>

        {/* APIs configuradas */}
        <Text style={{
          color: Colors.textSecondary, fontSize: 11,
          fontWeight: '700', letterSpacing: 1.2, marginBottom: 12,
        }}>
          CAPA DE DATOS
        </Text>
        <View style={{
          backgroundColor: Colors.surface,
          borderRadius: 16,
          borderWidth: 1,
          borderColor: Colors.border,
          padding: 16,
        }}>
          {[
            'data/storage/secure.storage.ts',
            'data/api/axios.client.ts',
            'data/api/api.error.ts',
            'data/api/auth.api.ts',
            'data/api/catalog.api.ts',
            'data/api/orders.api.ts',
            'data/api/users.api.ts',
          ].map((file, i, arr) => (
            <View key={file}>
              <View style={{
                flexDirection: 'row',
                justifyContent: 'space-between',
                alignItems: 'center',
                paddingVertical: 8,
              }}>
                <Text style={{
                  color: Colors.accent, fontSize: 11,
                  fontFamily: 'monospace', flex: 1,
                }}>
                  src/{file}
                </Text>
                <Text style={{color: Colors.success, fontWeight: '700'}}>✓</Text>
              </View>
              {i < arr.length - 1 && (
                <View style={{height: 0.5, backgroundColor: Colors.borderLight}} />
              )}
            </View>
          ))}
        </View>

      </ScrollView>
    </SafeAreaView>
  );
}
```

---

## ✅ Checkpoint Módulo 2

### Django debe estar corriendo

```bash
uv run python manage.py runserver 0.0.0.0:8000
```

### Poblar datos de prueba

```bash
uv run python manage.py shell -c "
from store.models import Category, Product
cat1 = Category.objects.get_or_create(name='Electrónica', slug='electronica', defaults={'description': 'Dispositivos', 'is_active': True})[0]
cat2 = Category.objects.get_or_create(name='Ropa', slug='ropa', defaults={'description': 'Moda', 'is_active': True})[0]
Product.objects.get_or_create(name='Laptop', defaults={'price': 999, 'stock': 5, 'category': cat1})
Product.objects.get_or_create(name='Camiseta', defaults={'price': 25, 'stock': 20, 'category': cat2})
print('Datos OK')
"
```

### Ejecutar la app

```bash
npx react-native run-android
# o
npx react-native run-ios
```

| Verificación | Resultado esperado |
|---|---|
| `✅ N categorías del backend` | Los datos reales de Django aparecen |
| Django apagado → error visible | `❌ Error de conexión` en rojo |
| 7 archivos de API con ✓ | Toda la capa de datos creada |
| Metro logs | `GET http://10.0.2.2:8000/api/categories/` visible |

### Ver logs de red en Metro

```bash
# En Android Studio → Logcat → filtrar "Axios"
# O en la consola de Metro aparecen los logs de la petición
```

---

## Resumen

| Elemento | Estado |
|---|---|
| `secure.storage.ts` — wrapper EncryptedStorage | ✅ |
| `ApiError` con parsing de errores Django | ✅ |
| `parseApiError` — normaliza todos los formatos | ✅ |
| `axios.client.ts` con interceptores JWT | ✅ |
| Renovación automática de token en 401 | ✅ |
| Cola de peticiones durante la renovación | ✅ |
| `authEvents` bus para 401 no recuperable | ✅ |
| `auth.api.ts` — login, register, logout | ✅ |
| `catalog.api.ts` — categorías y productos | ✅ |
| `orders.api.ts` — CRUD + checkout 3 pasos | ✅ |
| `users.api.ts` — CRUD admin de usuarios | ✅ |
| Verificación real con datos del backend | ✅ |

**Siguiente módulo →** M3: Auth — Login, Registro, AuthStore con Zustand y persistencia de sesión
