# ShopApp React Native — Módulo 13

## Testing — Unit tests con Jest + React Native Testing Library

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulos 1–12 completados.

---

> **Objetivo**
> Agregar una suite de tests automatizados para los stores de Zustand y los componentes más críticos de la app. Al finalizar, todos los tests deben pasar y la cobertura de código debe superar el 70%.

---

> **Checkpoint final**
> Al terminar este módulo deberás poder:
> - Ejecutar `npm test` y ver todos los tests en verde
> - Ejecutar `npm run test:coverage` y ver cobertura > 70%
> - Tener tests para: AuthStore, CatalogStore, CartStore, ProductCard, LoginScreen

---

## 13.1 Configuración de Jest

### Verificar configuración existente

React Native incluye Jest preconfigurado. Verifica que `package.json` tenga:

```json
{
  "scripts": {
    "test": "jest",
    "test:coverage": "jest --coverage",
    "test:watch": "jest --watch"
  },
  "jest": {
    "preset": "react-native",
    "setupFilesAfterEach": ["@testing-library/react-native/extend-expect"],
    "transformIgnorePatterns": [
      "node_modules/(?!(react-native|@react-native|react-native-fast-image|react-native-encrypted-storage|react-native-image-picker|zustand)/)"
    ],
    "collectCoverageFrom": [
      "src/**/*.{ts,tsx}",
      "!src/**/*.d.ts",
      "!src/**/index.ts",
      "!src/**/*.types.ts"
    ],
    "coverageThreshold": {
      "global": {
        "branches": 70,
        "functions": 70,
        "lines": 70,
        "statements": 70
      }
    }
  }
}
```

### Instalar dependencias de testing

```bash
npm install --save-dev \
  @testing-library/react-native \
  @testing-library/jest-native \
  @types/jest
```

### Archivo de setup

Crea `src/__tests__/setup.ts`:

```typescript
// src/__tests__/setup.ts
import '@testing-library/jest-native/extend-expect';

// Mock de react-native-encrypted-storage
jest.mock('react-native-encrypted-storage', () => ({
  setItem: jest.fn(() => Promise.resolve()),
  getItem: jest.fn(() => Promise.resolve(null)),
  removeItem: jest.fn(() => Promise.resolve()),
  clear: jest.fn(() => Promise.resolve()),
}));

// Mock de react-native-fast-image
jest.mock('react-native-fast-image', () => {
  const RN = jest.requireActual('react-native');
  const MockFastImage = ({ source, ...rest }: any) => {
    return <RN.Image source={source} {...rest} />;
  };
  MockFastImage.priority = { normal: 'normal', high: 'high', low: 'low' };
  MockFastImage.resizeMode = { cover: 'cover', contain: 'contain' };
  MockFastImage.cacheControl = { immutable: 'immutable', web: 'web' };
  return MockFastImage;
});

// Mock de react-native-image-picker
jest.mock('react-native-image-picker', () => ({
  launchImageLibrary: jest.fn(),
  launchCamera: jest.fn(),
}));

// Silenciar warnings de consola en tests
global.console.warn = jest.fn();
global.console.error = jest.fn();
```

Agrega `setupFilesAfterEach` en la config de Jest:

```json
"jest": {
  "preset": "react-native",
  "setupFilesAfterEach": [
    "@testing-library/jest-native/extend-expect",
    "./src/__tests__/setup.ts"
  ]
}
```

### Mock del cliente HTTP

Crea `src/__tests__/__mocks__/api.client.ts`:

```typescript
// Mock del módulo apiClient para todos los tests
const mockGet = jest.fn();
const mockPost = jest.fn();
const mockPatch = jest.fn();
const mockDelete = jest.fn();

export const apiClient = {
  get: mockGet,
  post: mockPost,
  patch: mockPatch,
  delete: mockDelete,
};

export { mockGet, mockPost, mockPatch, mockDelete };
```

---

## 13.2 Test del AuthStore

**Archivo:** `src/__tests__/stores/auth.store.test.ts`

```typescript
import { act } from 'react-test-renderer';
import EncryptedStorage from 'react-native-encrypted-storage';
import { apiClient } from '../__mocks__/api.client';

// Mock del módulo apiClient antes de importar el store
jest.mock('../../core/api/api.client', () => require('../__mocks__/api.client'));

// Importar DESPUÉS del mock
import { useAuthStore } from '../../features/auth/store/auth.store';

const mockUser = {
  id: 1,
  username: 'testuser',
  email: 'test@example.com',
  first_name: 'Test',
  last_name: 'User',
  is_staff: false,
  avatar_url: null,
};

describe('AuthStore', () => {
  beforeEach(() => {
    // Resetear el store al estado inicial antes de cada test
    useAuthStore.setState({
      user: null,
      token: null,
      isAuthenticated: false,
      isLoading: false,
      error: null,
    });
    jest.clearAllMocks();
  });

  describe('login', () => {
    it('debe autenticar al usuario y guardar el token en EncryptedStorage', async () => {
      // Arrange: mockear la respuesta exitosa del backend
      (apiClient.post as jest.Mock).mockResolvedValueOnce({
        data: {
          access: 'fake-access-token',
          refresh: 'fake-refresh-token',
          user: mockUser,
        },
      });

      // Act
      await act(async () => {
        await useAuthStore.getState().login('testuser', 'password123');
      });

      // Assert: el estado del store debe reflejar la autenticación
      const state = useAuthStore.getState();
      expect(state.isAuthenticated).toBe(true);
      expect(state.user).toEqual(mockUser);
      expect(state.token).toBe('fake-access-token');
      expect(state.error).toBeNull();

      // El token debe guardarse en EncryptedStorage
      expect(EncryptedStorage.setItem).toHaveBeenCalledWith(
        'access_token',
        'fake-access-token',
      );
    });

    it('debe establecer error cuando las credenciales son incorrectas', async () => {
      // Arrange: mockear respuesta 401 del backend
      (apiClient.post as jest.Mock).mockRejectedValueOnce({
        response: {
          status: 401,
          data: { detail: 'Credenciales inválidas' },
        },
      });

      // Act
      await act(async () => {
        await useAuthStore.getState().login('wronguser', 'wrongpass');
      });

      // Assert
      const state = useAuthStore.getState();
      expect(state.isAuthenticated).toBe(false);
      expect(state.user).toBeNull();
      expect(state.error).toBe('Credenciales inválidas');
    });

    it('debe activar isLoading durante la petición', async () => {
      let resolveLogin: (value: any) => void;
      const loginPromise = new Promise(resolve => { resolveLogin = resolve; });

      (apiClient.post as jest.Mock).mockReturnValueOnce(loginPromise);

      // Iniciar login sin esperar
      act(() => {
        useAuthStore.getState().login('testuser', 'pass');
      });

      // isLoading debe ser true mientras se procesa
      expect(useAuthStore.getState().isLoading).toBe(true);

      // Resolver la promesa
      await act(async () => {
        resolveLogin!({
          data: { access: 'token', refresh: 'refresh', user: mockUser },
        });
        await loginPromise;
      });

      expect(useAuthStore.getState().isLoading).toBe(false);
    });
  });

  describe('logout', () => {
    it('debe limpiar el estado y eliminar el token de EncryptedStorage', async () => {
      // Arrange: estado con usuario autenticado
      useAuthStore.setState({
        user: mockUser,
        token: 'some-token',
        isAuthenticated: true,
        isLoading: false,
        error: null,
      });

      // Act
      await act(async () => {
        await useAuthStore.getState().logout();
      });

      // Assert
      const state = useAuthStore.getState();
      expect(state.isAuthenticated).toBe(false);
      expect(state.user).toBeNull();
      expect(state.token).toBeNull();
      expect(EncryptedStorage.removeItem).toHaveBeenCalledWith('access_token');
    });
  });
});
```

---

## 13.3 Test del CatalogStore

**Archivo:** `src/__tests__/stores/catalog.store.test.ts`

```typescript
import { act } from 'react-test-renderer';

jest.mock('../../core/api/api.client', () => require('../__mocks__/api.client'));

import { useCatalogStore } from '../../features/catalog/store/catalog.store';
import { apiClient } from '../__mocks__/api.client';

const mockCategories = [
  { id: 1, name: 'Electrónica', slug: 'electronica', product_count: 5 },
  { id: 2, name: 'Ropa', slug: 'ropa', product_count: 12 },
];

const mockProductsResponse = {
  results: [
    {
      id: 1,
      name: 'Laptop Pro',
      slug: 'laptop-pro',
      description: 'Laptop de última generación',
      price: '1299.99',
      stock: 10,
      category: 1,
      category_name: 'Electrónica',
      sku: 'LAP-001',
      is_active: true,
      image_url: null,
      created_at: '2024-01-01T00:00:00Z',
    },
  ],
  count: 1,
};

describe('CatalogStore', () => {
  beforeEach(() => {
    useCatalogStore.setState({
      categories: [],
      products: [],
      selectedProduct: null,
      isLoading: false,
      error: null,
      searchQuery: '',
      selectedCategory: null,
      currentPage: 1,
      totalPages: 1,
    });
    jest.clearAllMocks();
  });

  describe('loadCategories', () => {
    it('debe cargar las categorías correctamente', async () => {
      (apiClient.get as jest.Mock).mockResolvedValueOnce({
        data: mockCategories,
      });

      await act(async () => {
        await useCatalogStore.getState().loadCategories();
      });

      const state = useCatalogStore.getState();
      expect(state.categories).toHaveLength(2);
      expect(state.categories[0].name).toBe('Electrónica');
      expect(state.isLoading).toBe(false);
      expect(state.error).toBeNull();
    });

    it('debe establecer error si la petición falla', async () => {
      (apiClient.get as jest.Mock).mockRejectedValueOnce({
        response: { data: { detail: 'Error del servidor' } },
      });

      await act(async () => {
        await useCatalogStore.getState().loadCategories();
      });

      expect(useCatalogStore.getState().error).toBe('Error del servidor');
    });
  });

  describe('loadProducts', () => {
    it('debe cargar productos y calcular páginas totales', async () => {
      (apiClient.get as jest.Mock).mockResolvedValueOnce({
        data: mockProductsResponse,
      });

      await act(async () => {
        await useCatalogStore.getState().loadProducts();
      });

      const state = useCatalogStore.getState();
      expect(state.products).toHaveLength(1);
      expect(state.products[0].name).toBe('Laptop Pro');
      expect(state.totalPages).toBe(1); // 1 producto / PAGE_SIZE
    });

    it('debe enviar parámetros de búsqueda y categoría a la API', async () => {
      (apiClient.get as jest.Mock).mockResolvedValueOnce({
        data: { results: [], count: 0 },
      });

      useCatalogStore.setState({
        searchQuery: 'laptop',
        selectedCategory: 1,
      } as any);

      await act(async () => {
        await useCatalogStore.getState().loadProducts();
      });

      expect(apiClient.get).toHaveBeenCalledWith(
        '/api/products/',
        expect.objectContaining({
          params: expect.objectContaining({
            search: 'laptop',
            category: 1,
          }),
        }),
      );
    });
  });
});
```

---

## 13.4 Test del CartStore

**Archivo:** `src/__tests__/stores/cart.store.test.ts`

```typescript
import { act } from 'react-test-renderer';
import { useCartStore } from '../../features/cart/store/cart.store';

const mockProduct = {
  id: 1,
  name: 'Laptop Pro',
  slug: 'laptop-pro',
  description: 'Test product',
  price: '1299.99',
  stock: 10,
  category: 1,
  category_name: 'Electrónica',
  sku: 'LAP-001',
  is_active: true,
  image_url: null,
  created_at: '2024-01-01T00:00:00Z',
};

describe('CartStore', () => {
  beforeEach(() => {
    // Limpiar el carrito antes de cada test
    useCartStore.setState({ items: [] });
  });

  describe('addItem', () => {
    it('debe agregar un producto nuevo al carrito', () => {
      act(() => {
        useCartStore.getState().addItem(mockProduct, 2);
      });

      const { items } = useCartStore.getState();
      expect(items).toHaveLength(1);
      expect(items[0].product.id).toBe(1);
      expect(items[0].quantity).toBe(2);
    });

    it('debe incrementar la cantidad si el producto ya existe en el carrito', () => {
      act(() => {
        useCartStore.getState().addItem(mockProduct, 1);
        useCartStore.getState().addItem(mockProduct, 3);
      });

      const { items } = useCartStore.getState();
      expect(items).toHaveLength(1); // Solo un item, no duplicado
      expect(items[0].quantity).toBe(4); // 1 + 3
    });

    it('no debe exceder el stock disponible al agregar', () => {
      act(() => {
        // El producto tiene stock: 10
        useCartStore.getState().addItem(mockProduct, 8);
        useCartStore.getState().addItem(mockProduct, 5); // Total sería 13, mayor al stock
      });

      const { items } = useCartStore.getState();
      // La cantidad debe estar limitada al stock (10)
      expect(items[0].quantity).toBeLessThanOrEqual(mockProduct.stock);
    });
  });

  describe('removeItem', () => {
    it('debe eliminar un producto del carrito', () => {
      act(() => {
        useCartStore.getState().addItem(mockProduct, 2);
        useCartStore.getState().removeItem(1);
      });

      expect(useCartStore.getState().items).toHaveLength(0);
    });

    it('no debe afectar otros items al eliminar uno', () => {
      const secondProduct = { ...mockProduct, id: 2, name: 'Mouse' };

      act(() => {
        useCartStore.getState().addItem(mockProduct, 1);
        useCartStore.getState().addItem(secondProduct, 1);
        useCartStore.getState().removeItem(1);
      });

      const { items } = useCartStore.getState();
      expect(items).toHaveLength(1);
      expect(items[0].product.id).toBe(2);
    });
  });

  describe('total', () => {
    it('debe calcular el total correctamente', () => {
      const secondProduct = {
        ...mockProduct,
        id: 2,
        name: 'Mouse',
        price: '25.00',
      };

      act(() => {
        useCartStore.getState().addItem(mockProduct, 1);  // 1299.99
        useCartStore.getState().addItem(secondProduct, 2); // 50.00
      });

      const total = useCartStore.getState().getTotal();
      // 1299.99 + (25.00 * 2) = 1349.99
      expect(total).toBeCloseTo(1349.99, 2);
    });

    it('debe retornar 0 para un carrito vacío', () => {
      const total = useCartStore.getState().getTotal();
      expect(total).toBe(0);
    });
  });

  describe('updateQuantity', () => {
    it('debe actualizar la cantidad de un item', () => {
      act(() => {
        useCartStore.getState().addItem(mockProduct, 1);
        useCartStore.getState().updateQuantity(1, 5);
      });

      expect(useCartStore.getState().items[0].quantity).toBe(5);
    });
  });

  describe('clearCart', () => {
    it('debe vaciar el carrito completamente', () => {
      act(() => {
        useCartStore.getState().addItem(mockProduct, 3);
        useCartStore.getState().clearCart();
      });

      expect(useCartStore.getState().items).toHaveLength(0);
    });
  });
});
```

---

## 13.5 Test de componentes

**Archivo:** `src/__tests__/components/ProductCard.test.tsx`

```typescript
import React from 'react';
import { render, fireEvent, screen } from '@testing-library/react-native';
import { ProductCard } from '../../features/catalog/components/ProductCard';

const mockProduct = {
  id: 1,
  name: 'Laptop Pro',
  slug: 'laptop-pro',
  description: 'Una laptop increíble',
  price: '1299.99',
  stock: 10,
  category: 1,
  category_name: 'Electrónica',
  sku: 'LAP-001',
  is_active: true,
  image_url: null,
  created_at: '2024-01-01T00:00:00Z',
};

describe('ProductCard', () => {
  it('debe renderizar el nombre y precio del producto', () => {
    render(
      <ProductCard product={mockProduct} onPress={jest.fn()} />,
    );

    expect(screen.getByText('Laptop Pro')).toBeTruthy();
    expect(screen.getByText('$ 1299.99')).toBeTruthy();
  });

  it('debe mostrar la categoría del producto', () => {
    render(
      <ProductCard product={mockProduct} onPress={jest.fn()} />,
    );

    expect(screen.getByText('Electrónica')).toBeTruthy();
  });

  it('debe llamar onPress cuando se presiona la tarjeta', () => {
    const mockOnPress = jest.fn();

    render(
      <ProductCard product={mockProduct} onPress={mockOnPress} />,
    );

    // Obtener el elemento TouchableOpacity por su rol
    const card = screen.getByRole('button');
    fireEvent.press(card);

    expect(mockOnPress).toHaveBeenCalledTimes(1);
    expect(mockOnPress).toHaveBeenCalledWith(mockProduct);
  });

  it('debe mostrar "Sin stock" cuando stock es 0', () => {
    const outOfStockProduct = { ...mockProduct, stock: 0 };
    render(
      <ProductCard product={outOfStockProduct} onPress={jest.fn()} />,
    );

    expect(screen.getByText(/sin stock/i)).toBeTruthy();
  });

  it('debe mostrar el placeholder cuando no hay imagen', () => {
    render(
      <ProductCard product={mockProduct} onPress={jest.fn()} />,
    );

    // El placeholder muestra el texto "Sin imagen"
    expect(screen.getByText('Sin imagen')).toBeTruthy();
  });
});
```

---

## 13.6 Test de pantallas

**Archivo:** `src/__tests__/screens/LoginScreen.test.tsx`

```typescript
import React from 'react';
import { render, fireEvent, waitFor, screen } from '@testing-library/react-native';
import { NavigationContainer } from '@react-navigation/native';

// Mock del store de autenticación
const mockLogin = jest.fn();
jest.mock('../../features/auth/store/auth.store', () => ({
  useAuthStore: () => ({
    login: mockLogin,
    isLoading: false,
    error: null,
  }),
}));

import { LoginScreen } from '../../features/auth/screens/LoginScreen';

// Helper para renderizar con NavigationContainer (requerido por React Navigation)
const renderWithNavigation = (component: React.ReactElement) => {
  return render(
    <NavigationContainer>
      {component}
    </NavigationContainer>,
  );
};

describe('LoginScreen', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('debe renderizar los campos de usuario y contraseña', () => {
    renderWithNavigation(<LoginScreen />);

    expect(screen.getByPlaceholderText(/usuario/i)).toBeTruthy();
    expect(screen.getByPlaceholderText(/contraseña/i)).toBeTruthy();
  });

  it('debe llamar a store.login con los datos ingresados al presionar el botón', async () => {
    mockLogin.mockResolvedValueOnce(undefined);
    renderWithNavigation(<LoginScreen />);

    // Ingresar datos en el formulario
    fireEvent.changeText(
      screen.getByPlaceholderText(/usuario/i),
      'admin',
    );
    fireEvent.changeText(
      screen.getByPlaceholderText(/contraseña/i),
      'secret123',
    );

    // Presionar el botón de login
    fireEvent.press(screen.getByText(/iniciar sesión/i));

    await waitFor(() => {
      expect(mockLogin).toHaveBeenCalledWith('admin', 'secret123');
    });
  });

  it('debe mostrar un error de validación si los campos están vacíos', async () => {
    renderWithNavigation(<LoginScreen />);

    // Presionar login sin llenar campos
    fireEvent.press(screen.getByText(/iniciar sesión/i));

    // El store.login NO debe llamarse
    expect(mockLogin).not.toHaveBeenCalled();
  });

  it('debe deshabilitar el botón mientras isLoading es true', () => {
    jest.mock('../../features/auth/store/auth.store', () => ({
      useAuthStore: () => ({
        login: mockLogin,
        isLoading: true, // Loading activo
        error: null,
      }),
    }));

    renderWithNavigation(<LoginScreen />);
    const button = screen.getByText(/iniciar sesión/i);

    // El botón debe estar deshabilitado
    expect(button.props.disabled ?? false).toBe(true);
  });
});
```

---

## 13.7 Ejecutar los tests

### Correr todos los tests

```bash
npm test
```

Salida esperada:
```
PASS src/__tests__/stores/auth.store.test.ts
PASS src/__tests__/stores/catalog.store.test.ts
PASS src/__tests__/stores/cart.store.test.ts
PASS src/__tests__/components/ProductCard.test.tsx
PASS src/__tests__/screens/LoginScreen.test.tsx

Test Suites: 5 passed, 5 total
Tests:       22 passed, 22 total
```

### Ver reporte de cobertura

```bash
npm run test:coverage
```

Esto genera un directorio `coverage/` con el reporte HTML. Abre `coverage/lcov-report/index.html` en el navegador para ver el reporte visual por archivo.

### Correr un archivo específico

```bash
npx jest src/__tests__/stores/auth.store.test.ts
npx jest --testPathPattern="auth"   # Todos los tests que contengan "auth"
```

### Modo watch (durante desarrollo)

```bash
npm run test:watch
# Presiona 'p' para filtrar por nombre de archivo
# Presiona 'f' para correr solo tests fallidos
```

### Solución de problemas comunes

| Error | Causa | Solución |
|-------|-------|----------|
| `Cannot find module 'react-native-fast-image'` | Mock no configurado | Verificar `setup.ts` y `transformIgnorePatterns` |
| `useAuthStore is not a function` | Import antes del mock | Mover imports después de `jest.mock()` |
| `act()` warning en consola | Actualización de estado sin envolver | Envolver en `act(async () => { ... })` |
| `NavigationContainer` error en pantallas | Pantalla sin contexto de navegación | Usar helper `renderWithNavigation` |

---

> **Checkpoint — Módulo 13**
>
> Verifica que puedes:
> - [ ] Ejecutar `npm test` y ver todos los tests en verde (sin errores en rojo)
> - [ ] Ejecutar `npm run test:coverage` y ver cobertura global > 70%
> - [ ] Los tests del AuthStore verifican login exitoso, login fallido y logout
> - [ ] Los tests del CartStore verifican addItem, removeItem, cálculo de total y límite de stock
> - [ ] El test de ProductCard verifica renderizado, press y estados especiales (sin stock, sin imagen)
> - [ ] El test de LoginScreen verifica que se llama a `store.login` con las credenciales correctas

---

**Siguiente módulo:** Módulo 14 — Pulido y Optimización: Loading skeletons, error boundaries y UX
