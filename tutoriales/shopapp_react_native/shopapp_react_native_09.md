# ShopApp React Native — Módulo 9

## Panel Admin — Gestión de Catálogo (Productos y Categorías)

**Objetivo:** Implementar un panel de administración accesible únicamente para usuarios con `is_staff: true`. Permite crear, editar, eliminar y activar/desactivar productos y categorías del catálogo directamente desde la app móvil.

> **Checkpoint final:** Un usuario staff puede ver el catálogo de administración, crear un nuevo producto, editar su precio, activar/desactivar con toggle, y gestionar categorías. Usuarios normales no ven la pestaña de admin ni pueden acceder a estas rutas.

---

## 9.1 Guard de admin

**Archivo:** `src/presentation/navigation/AdminGuard.tsx`

Componente de protección de rutas que verifica si el usuario autenticado tiene el flag `is_staff`. Si no lo tiene, muestra una pantalla de acceso denegado en lugar del contenido protegido.

```typescript
// src/presentation/navigation/AdminGuard.tsx

import React from 'react';
import { View, Text, StyleSheet, TouchableOpacity } from 'react-native';
import { useNavigation } from '@react-navigation/native';
import { Colors } from '../../constants/Colors';
import { useAuthStore } from '../../domain/store/auth.store';

interface AdminGuardProps {
  children: React.ReactNode;
}

// Pantalla de acceso denegado para usuarios sin permisos
function AccessDeniedScreen() {
  const navigation = useNavigation();

  return (
    <View style={guardStyles.container}>
      <Text style={guardStyles.icon}>🔒</Text>
      <Text style={guardStyles.title}>Acceso restringido</Text>
      <Text style={guardStyles.subtitle}>
        Esta sección es exclusiva para administradores.{'\n'}
        Tu cuenta no tiene los permisos necesarios.
      </Text>
      <TouchableOpacity
        style={guardStyles.btn}
        onPress={() => navigation.goBack()}
        activeOpacity={0.8}
      >
        <Text style={guardStyles.btnText}>Volver</Text>
      </TouchableOpacity>
    </View>
  );
}

const guardStyles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
    alignItems: 'center',
    justifyContent: 'center',
    padding: 32,
  },
  icon: {
    fontSize: 56,
    marginBottom: 20,
  },
  title: {
    fontSize: 22,
    fontWeight: '800',
    color: Colors.textPrimary,
    textAlign: 'center',
    marginBottom: 12,
  },
  subtitle: {
    fontSize: 14,
    color: Colors.textSecondary,
    textAlign: 'center',
    lineHeight: 22,
    marginBottom: 32,
  },
  btn: {
    backgroundColor: Colors.surface2,
    borderRadius: 12,
    paddingHorizontal: 28,
    paddingVertical: 12,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  btnText: {
    color: Colors.textPrimary,
    fontSize: 15,
    fontWeight: '600',
  },
});

// Guard principal: renderiza hijos solo si el usuario es staff
export default function AdminGuard({ children }: AdminGuardProps) {
  const user = useAuthStore((state) => state.user);

  if (!user?.isStaff) {
    return <AccessDeniedScreen />;
  }

  return <>{children}</>;
}

// Hook de conveniencia para verificar permisos en componentes
export function useIsAdmin(): boolean {
  return useAuthStore((state) => state.user?.isStaff ?? false);
}
```

**Uso del guard en una pantalla:**

```typescript
// Ejemplo: envolver una pantalla con AdminGuard
export default function AdminProductsScreen() {
  return (
    <AdminGuard>
      <AdminProductsContent />
    </AdminGuard>
  );
}
```

> Asegúrate de que `AuthStore.user` tenga un campo `isStaff: boolean` mapeado desde `is_staff` de la API de Django.

---

## 9.2 AdminCatalogStore

**Archivo:** `src/domain/store/admin.catalog.store.ts`

Store de Zustand con el estado completo del catálogo de administración: productos y categorías. Expone todas las acciones CRUD para ambas entidades.

```typescript
// src/domain/store/admin.catalog.store.ts

import { create } from 'zustand';
import { apiClient } from '../../data/api/apiClient';

// Tipos locales del panel admin
export interface AdminCategory {
  id: number;
  name: string;
  slug: string;
  product_count?: number;
}

export interface AdminProduct {
  id: number;
  name: string;
  description: string;
  price: string;
  stock: number;
  is_active: boolean;
  category: AdminCategory | null;
  category_id: number | null;
  image_url: string | null;
}

export interface ProductFormData {
  name: string;
  description: string;
  price: string;
  stock: number;
  category_id: number | null;
  is_active: boolean;
}

export interface CategoryFormData {
  name: string;
  slug?: string;
}

interface AdminCatalogState {
  // Estado
  products: AdminProduct[];
  categories: AdminCategory[];
  isLoading: boolean;
  isSaving: boolean;
  error: string | null;

  // Acciones de productos
  loadProducts: () => Promise<void>;
  createProduct: (data: ProductFormData) => Promise<AdminProduct>;
  updateProduct: (id: number, data: Partial<ProductFormData>) => Promise<AdminProduct>;
  deleteProduct: (id: number) => Promise<void>;
  toggleProductActive: (id: number) => Promise<void>;

  // Acciones de categorías
  loadCategories: () => Promise<void>;
  createCategory: (data: CategoryFormData) => Promise<AdminCategory>;
  updateCategory: (id: number, data: CategoryFormData) => Promise<AdminCategory>;
  deleteCategory: (id: number) => Promise<void>;

  // Utilidades
  clearError: () => void;
}

export const useAdminCatalogStore = create<AdminCatalogState>((set, get) => ({
  products: [],
  categories: [],
  isLoading: false,
  isSaving: false,
  error: null,

  // ─── Productos ───────────────────────────────────────────────

  loadProducts: async () => {
    set({ isLoading: true, error: null });
    try {
      // Asume endpoint paginado; adaptar si la API devuelve lista directa
      const response = await apiClient.get<{ results: AdminProduct[] } | AdminProduct[]>(
        '/api/admin/products/',
      );
      const products = Array.isArray(response.data)
        ? response.data
        : response.data.results;
      set({ products, isLoading: false });
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al cargar productos',
        isLoading: false,
      });
    }
  },

  createProduct: async (data: ProductFormData) => {
    set({ isSaving: true, error: null });
    try {
      const response = await apiClient.post<AdminProduct>(
        '/api/admin/products/',
        data,
      );
      const newProduct = response.data;
      set((state) => ({
        products: [newProduct, ...state.products],
        isSaving: false,
      }));
      return newProduct;
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al crear el producto',
        isSaving: false,
      });
      throw err;
    }
  },

  updateProduct: async (id: number, data: Partial<ProductFormData>) => {
    set({ isSaving: true, error: null });
    try {
      const response = await apiClient.patch<AdminProduct>(
        `/api/admin/products/${id}/`,
        data,
      );
      const updated = response.data;
      set((state) => ({
        products: state.products.map((p) => (p.id === id ? updated : p)),
        isSaving: false,
      }));
      return updated;
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al actualizar el producto',
        isSaving: false,
      });
      throw err;
    }
  },

  deleteProduct: async (id: number) => {
    try {
      await apiClient.delete(`/api/admin/products/${id}/`);
      set((state) => ({
        products: state.products.filter((p) => p.id !== id),
      }));
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al eliminar el producto',
      });
      throw err;
    }
  },

  // Toggle optimista: actualiza el estado local inmediatamente
  // y revierte si la petición falla
  toggleProductActive: async (id: number) => {
    const product = get().products.find((p) => p.id === id);
    if (!product) return;

    // Actualización optimista
    set((state) => ({
      products: state.products.map((p) =>
        p.id === id ? { ...p, is_active: !p.is_active } : p,
      ),
    }));

    try {
      await apiClient.patch(`/api/admin/products/${id}/`, {
        is_active: !product.is_active,
      });
    } catch (err: any) {
      // Revertir en caso de error
      set((state) => ({
        products: state.products.map((p) =>
          p.id === id ? { ...p, is_active: product.is_active } : p,
        ),
        error: 'No se pudo cambiar el estado del producto',
      }));
    }
  },

  // ─── Categorías ──────────────────────────────────────────────

  loadCategories: async () => {
    set({ isLoading: true, error: null });
    try {
      const response = await apiClient.get<AdminCategory[]>(
        '/api/admin/categories/',
      );
      set({ categories: response.data, isLoading: false });
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al cargar categorías',
        isLoading: false,
      });
    }
  },

  createCategory: async (data: CategoryFormData) => {
    set({ isSaving: true, error: null });
    try {
      const response = await apiClient.post<AdminCategory>(
        '/api/admin/categories/',
        data,
      );
      const newCat = response.data;
      set((state) => ({
        categories: [...state.categories, newCat],
        isSaving: false,
      }));
      return newCat;
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al crear la categoría',
        isSaving: false,
      });
      throw err;
    }
  },

  updateCategory: async (id: number, data: CategoryFormData) => {
    set({ isSaving: true, error: null });
    try {
      const response = await apiClient.patch<AdminCategory>(
        `/api/admin/categories/${id}/`,
        data,
      );
      const updated = response.data;
      set((state) => ({
        categories: state.categories.map((c) => (c.id === id ? updated : c)),
        isSaving: false,
      }));
      return updated;
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al actualizar la categoría',
        isSaving: false,
      });
      throw err;
    }
  },

  deleteCategory: async (id: number) => {
    try {
      await apiClient.delete(`/api/admin/categories/${id}/`);
      set((state) => ({
        categories: state.categories.filter((c) => c.id !== id),
      }));
    } catch (err: any) {
      set({
        error: err.response?.data?.detail ?? 'Error al eliminar la categoría',
      });
      throw err;
    }
  },

  clearError: () => set({ error: null }),
}));
```

**Patrón de actualización optimista en `toggleProductActive`:**

```
1. Usuario toca el Switch → estado local cambia inmediatamente (UI responde al instante)
2. Se envía PATCH a la API en background
3. Si la API responde OK → el estado optimista ya es correcto, no se hace nada
4. Si la API falla → se revierte al valor original y se muestra el error
```

---

## 9.3 AdminProductsScreen

**Archivo:** `src/presentation/screens/admin/AdminProductsScreen.tsx`

Lista todos los productos con nombre, precio, stock y toggle de `is_active`. Incluye FAB para añadir y soporte para editar/eliminar mediante long press.

```typescript
// src/presentation/screens/admin/AdminProductsScreen.tsx

import React, { useEffect, useCallback, useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  FlatList,
  Switch,
  TouchableOpacity,
  ActivityIndicator,
  Alert,
  RefreshControl,
  ListRenderItemInfo,
} from 'react-native';
import { Colors } from '../../../constants/Colors';
import {
  useAdminCatalogStore,
  AdminProduct,
} from '../../../domain/store/admin.catalog.store';
import AdminGuard from '../../navigation/AdminGuard';
import ProductFormModal from './components/ProductFormModal';

function ProductRow({
  product,
  onToggle,
  onEdit,
  onDelete,
}: {
  product: AdminProduct;
  onToggle: () => void;
  onEdit: () => void;
  onDelete: () => void;
}) {
  const handleLongPress = () => {
    Alert.alert(
      product.name,
      'Selecciona una acción',
      [
        { text: 'Editar', onPress: onEdit },
        { text: 'Eliminar', style: 'destructive', onPress: onDelete },
        { text: 'Cancelar', style: 'cancel' },
      ],
    );
  };

  return (
    <TouchableOpacity
      style={rowStyles.row}
      onPress={onEdit}
      onLongPress={handleLongPress}
      activeOpacity={0.75}
      delayLongPress={400}
    >
      {/* Estado activo/inactivo */}
      <View
        style={[
          rowStyles.activeDot,
          { backgroundColor: product.is_active ? Colors.success : Colors.error },
        ]}
      />

      {/* Información del producto */}
      <View style={rowStyles.info}>
        <Text style={rowStyles.name} numberOfLines={1}>
          {product.name}
        </Text>
        <Text style={rowStyles.meta}>
          Stock: {product.stock} · {product.category?.name ?? 'Sin categoría'}
        </Text>
      </View>

      {/* Precio */}
      <Text style={rowStyles.price}>
        ${parseFloat(product.price).toFixed(2)}
      </Text>

      {/* Toggle is_active */}
      <Switch
        value={product.is_active}
        onValueChange={onToggle}
        thumbColor={Colors.textPrimary}
        trackColor={{ false: Colors.surface2, true: Colors.accent }}
        style={rowStyles.switch}
      />
    </TouchableOpacity>
  );
}

const rowStyles = StyleSheet.create({
  row: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: Colors.surface,
    borderRadius: 12,
    padding: 14,
    marginBottom: 8,
    borderWidth: 1,
    borderColor: Colors.border,
    gap: 10,
  },
  activeDot: {
    width: 8,
    height: 8,
    borderRadius: 4,
    flexShrink: 0,
  },
  info: {
    flex: 1,
  },
  name: {
    fontSize: 14,
    fontWeight: '600',
    color: Colors.textPrimary,
    marginBottom: 2,
  },
  meta: {
    fontSize: 12,
    color: Colors.textSecondary,
  },
  price: {
    fontSize: 14,
    fontWeight: '700',
    color: Colors.accent,
    minWidth: 60,
    textAlign: 'right',
  },
  switch: {
    transform: [{ scaleX: 0.85 }, { scaleY: 0.85 }],
  },
});

function AdminProductsContent() {
  const {
    products,
    isLoading,
    deleteProduct,
    toggleProductActive,
    loadProducts,
  } = useAdminCatalogStore();

  const [modalVisible, setModalVisible] = useState(false);
  const [editingProduct, setEditingProduct] = useState<AdminProduct | null>(null);

  useEffect(() => {
    loadProducts();
  }, []);

  const handleRefresh = useCallback(() => {
    loadProducts();
  }, []);

  const handleDelete = (product: AdminProduct) => {
    Alert.alert(
      'Eliminar producto',
      `¿Confirmas que deseas eliminar "${product.name}"? Esta acción no se puede deshacer.`,
      [
        { text: 'Cancelar', style: 'cancel' },
        {
          text: 'Eliminar',
          style: 'destructive',
          onPress: async () => {
            try {
              await deleteProduct(product.id);
            } catch {
              Alert.alert('Error', 'No se pudo eliminar el producto.');
            }
          },
        },
      ],
    );
  };

  const handleOpenCreate = () => {
    setEditingProduct(null);
    setModalVisible(true);
  };

  const handleOpenEdit = (product: AdminProduct) => {
    setEditingProduct(product);
    setModalVisible(true);
  };

  const renderItem = ({ item }: ListRenderItemInfo<AdminProduct>) => (
    <ProductRow
      product={item}
      onToggle={() => toggleProductActive(item.id)}
      onEdit={() => handleOpenEdit(item)}
      onDelete={() => handleDelete(item)}
    />
  );

  return (
    <View style={screenStyles.container}>
      {isLoading && products.length === 0 ? (
        <View style={screenStyles.center}>
          <ActivityIndicator size="large" color={Colors.accent} />
        </View>
      ) : (
        <FlatList
          data={products}
          keyExtractor={(item) => String(item.id)}
          renderItem={renderItem}
          contentContainerStyle={screenStyles.list}
          ListHeaderComponent={
            <View style={screenStyles.listHeader}>
              <Text style={screenStyles.count}>
                {products.length} productos
              </Text>
              <Text style={screenStyles.hint}>
                Mantén pulsado para más opciones
              </Text>
            </View>
          }
          ListEmptyComponent={
            <View style={screenStyles.empty}>
              <Text style={screenStyles.emptyText}>
                No hay productos.{'\n'}Presiona + para añadir uno.
              </Text>
            </View>
          }
          refreshControl={
            <RefreshControl
              refreshing={isLoading}
              onRefresh={handleRefresh}
              tintColor={Colors.accent}
            />
          }
          showsVerticalScrollIndicator={false}
        />
      )}

      {/* FAB — botón flotante para añadir producto */}
      <TouchableOpacity
        style={screenStyles.fab}
        onPress={handleOpenCreate}
        activeOpacity={0.85}
      >
        <Text style={screenStyles.fabIcon}>+</Text>
      </TouchableOpacity>

      {/* Modal de formulario */}
      <ProductFormModal
        visible={modalVisible}
        product={editingProduct}
        onClose={() => setModalVisible(false)}
        onSuccess={() => {
          setModalVisible(false);
          loadProducts();
        }}
      />
    </View>
  );
}

export default function AdminProductsScreen() {
  return (
    <AdminGuard>
      <AdminProductsContent />
    </AdminGuard>
  );
}

const screenStyles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  center: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
  },
  list: {
    padding: 16,
    paddingBottom: 90,
  },
  listHeader: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 12,
  },
  count: {
    fontSize: 13,
    color: Colors.textSecondary,
    fontWeight: '500',
  },
  hint: {
    fontSize: 11,
    color: Colors.textSecondary,
    fontStyle: 'italic',
  },
  empty: {
    alignItems: 'center',
    paddingTop: 60,
  },
  emptyText: {
    fontSize: 15,
    color: Colors.textSecondary,
    textAlign: 'center',
    lineHeight: 24,
  },
  fab: {
    position: 'absolute',
    bottom: 24,
    right: 20,
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: Colors.accent,
    alignItems: 'center',
    justifyContent: 'center',
    shadowColor: Colors.accent,
    shadowOpacity: 0.5,
    shadowRadius: 12,
    shadowOffset: { width: 0, height: 4 },
    elevation: 8,
  },
  fabIcon: {
    fontSize: 28,
    color: Colors.textPrimary,
    fontWeight: '300',
    lineHeight: 32,
  },
});
```

---

## 9.4 ProductFormModal

**Archivo:** `src/presentation/screens/admin/components/ProductFormModal.tsx`

Modal tipo bottom sheet con el formulario de producto. Sirve tanto para crear (cuando `product` es `null`) como para editar (cuando `product` tiene datos).

```typescript
// src/presentation/screens/admin/components/ProductFormModal.tsx

import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  StyleSheet,
  Modal,
  TextInput,
  TouchableOpacity,
  ScrollView,
  ActivityIndicator,
  Alert,
  KeyboardAvoidingView,
  Platform,
  Switch,
} from 'react-native';
import { Colors } from '../../../../constants/Colors';
import {
  useAdminCatalogStore,
  AdminProduct,
  ProductFormData,
} from '../../../../domain/store/admin.catalog.store';

interface ProductFormModalProps {
  visible: boolean;
  product: AdminProduct | null; // null = modo creación
  onClose: () => void;
  onSuccess: () => void;
}

function FormInput({
  label,
  value,
  onChangeText,
  placeholder,
  keyboardType,
  multiline,
  numberOfLines,
}: {
  label: string;
  value: string;
  onChangeText: (t: string) => void;
  placeholder?: string;
  keyboardType?: 'default' | 'numeric' | 'decimal-pad';
  multiline?: boolean;
  numberOfLines?: number;
}) {
  return (
    <View style={inputStyles.wrapper}>
      <Text style={inputStyles.label}>{label}</Text>
      <TextInput
        style={[inputStyles.input, multiline && inputStyles.multiline]}
        value={value}
        onChangeText={onChangeText}
        placeholder={placeholder ?? label}
        placeholderTextColor={Colors.textSecondary}
        keyboardType={keyboardType ?? 'default'}
        multiline={multiline}
        numberOfLines={numberOfLines}
        autoCorrect={false}
      />
    </View>
  );
}

const inputStyles = StyleSheet.create({
  wrapper: {
    marginBottom: 14,
  },
  label: {
    fontSize: 11,
    fontWeight: '700',
    color: Colors.textSecondary,
    textTransform: 'uppercase',
    letterSpacing: 0.6,
    marginBottom: 5,
  },
  input: {
    backgroundColor: Colors.surface2,
    borderRadius: 10,
    borderWidth: 1,
    borderColor: Colors.border,
    paddingHorizontal: 12,
    paddingVertical: 10,
    fontSize: 15,
    color: Colors.textPrimary,
  },
  multiline: {
    height: 80,
    textAlignVertical: 'top',
    paddingTop: 10,
  },
});

export default function ProductFormModal({
  visible,
  product,
  onClose,
  onSuccess,
}: ProductFormModalProps) {
  const { categories, createProduct, updateProduct, isSaving } =
    useAdminCatalogStore();

  const isEditing = product !== null;

  // Estado del formulario
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');
  const [price, setPrice] = useState('');
  const [stock, setStock] = useState('');
  const [categoryId, setCategoryId] = useState<number | null>(null);
  const [isActive, setIsActive] = useState(true);

  // Precargar valores al abrir en modo edición
  useEffect(() => {
    if (product) {
      setName(product.name);
      setDescription(product.description);
      setPrice(product.price);
      setStock(String(product.stock));
      setCategoryId(product.category_id);
      setIsActive(product.is_active);
    } else {
      // Restablecer para modo creación
      setName('');
      setDescription('');
      setPrice('');
      setStock('0');
      setCategoryId(categories[0]?.id ?? null);
      setIsActive(true);
    }
  }, [product, visible]);

  const validateAndGetData = (): ProductFormData | null => {
    if (!name.trim()) {
      Alert.alert('Campo requerido', 'El nombre del producto es obligatorio.');
      return null;
    }
    const parsedPrice = parseFloat(price);
    if (isNaN(parsedPrice) || parsedPrice < 0) {
      Alert.alert('Precio inválido', 'Ingresa un precio numérico válido.');
      return null;
    }
    const parsedStock = parseInt(stock, 10);
    if (isNaN(parsedStock) || parsedStock < 0) {
      Alert.alert('Stock inválido', 'Ingresa un valor de stock válido.');
      return null;
    }
    return {
      name: name.trim(),
      description: description.trim(),
      price: parsedPrice.toFixed(2),
      stock: parsedStock,
      category_id: categoryId,
      is_active: isActive,
    };
  };

  const handleSubmit = async () => {
    const data = validateAndGetData();
    if (!data) return;

    try {
      if (isEditing && product) {
        await updateProduct(product.id, data);
        Alert.alert('Producto actualizado', `"${data.name}" ha sido actualizado.`);
      } else {
        await createProduct(data);
        Alert.alert('Producto creado', `"${data.name}" ha sido añadido al catálogo.`);
      }
      onSuccess();
    } catch {
      Alert.alert(
        'Error',
        isEditing
          ? 'No se pudo actualizar el producto.'
          : 'No se pudo crear el producto.',
      );
    }
  };

  return (
    <Modal
      visible={visible}
      animationType="slide"
      transparent
      onRequestClose={onClose}
    >
      <View style={modalStyles.overlay}>
        <KeyboardAvoidingView
          behavior={Platform.OS === 'ios' ? 'padding' : undefined}
          style={{ width: '100%' }}
        >
          <View style={modalStyles.sheet}>
            {/* Handle visual */}
            <View style={modalStyles.handle} />

            {/* Cabecera del modal */}
            <View style={modalStyles.header}>
              <Text style={modalStyles.title}>
                {isEditing ? 'Editar producto' : 'Nuevo producto'}
              </Text>
              <TouchableOpacity onPress={onClose} hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}>
                <Text style={modalStyles.closeBtn}>✕</Text>
              </TouchableOpacity>
            </View>

            <ScrollView
              showsVerticalScrollIndicator={false}
              keyboardShouldPersistTaps="handled"
            >
              <FormInput
                label="Nombre"
                value={name}
                onChangeText={setName}
                placeholder="Nombre del producto"
              />
              <FormInput
                label="Descripción"
                value={description}
                onChangeText={setDescription}
                placeholder="Descripción opcional"
                multiline
                numberOfLines={3}
              />
              <FormInput
                label="Precio"
                value={price}
                onChangeText={setPrice}
                placeholder="0.00"
                keyboardType="decimal-pad"
              />
              <FormInput
                label="Stock"
                value={stock}
                onChangeText={setStock}
                placeholder="0"
                keyboardType="numeric"
              />

              {/* Selector de categoría */}
              <View style={modalStyles.fieldWrapper}>
                <Text style={modalStyles.fieldLabel}>Categoría</Text>
                <ScrollView
                  horizontal
                  showsHorizontalScrollIndicator={false}
                  style={modalStyles.categoryScroll}
                >
                  {categories.map((cat) => (
                    <TouchableOpacity
                      key={cat.id}
                      style={[
                        modalStyles.categoryChip,
                        categoryId === cat.id && modalStyles.categoryChipActive,
                      ]}
                      onPress={() => setCategoryId(cat.id)}
                    >
                      <Text
                        style={[
                          modalStyles.categoryChipText,
                          categoryId === cat.id &&
                            modalStyles.categoryChipTextActive,
                        ]}
                      >
                        {cat.name}
                      </Text>
                    </TouchableOpacity>
                  ))}
                </ScrollView>
              </View>

              {/* Toggle is_active */}
              <View style={modalStyles.toggleRow}>
                <View>
                  <Text style={modalStyles.toggleLabel}>Producto activo</Text>
                  <Text style={modalStyles.toggleHint}>
                    {isActive
                      ? 'Visible en el catálogo'
                      : 'Oculto para los clientes'}
                  </Text>
                </View>
                <Switch
                  value={isActive}
                  onValueChange={setIsActive}
                  thumbColor={Colors.textPrimary}
                  trackColor={{ false: Colors.surface2, true: Colors.accent }}
                />
              </View>

              {/* Botón de acción */}
              <TouchableOpacity
                style={[
                  modalStyles.submitBtn,
                  isSaving && modalStyles.submitBtnDisabled,
                ]}
                onPress={handleSubmit}
                disabled={isSaving}
                activeOpacity={0.8}
              >
                {isSaving ? (
                  <ActivityIndicator color={Colors.textPrimary} />
                ) : (
                  <Text style={modalStyles.submitBtnText}>
                    {isEditing ? 'Guardar cambios' : 'Crear producto'}
                  </Text>
                )}
              </TouchableOpacity>
            </ScrollView>
          </View>
        </KeyboardAvoidingView>
      </View>
    </Modal>
  );
}

const modalStyles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.65)',
    justifyContent: 'flex-end',
  },
  sheet: {
    backgroundColor: Colors.surface,
    borderTopLeftRadius: 24,
    borderTopRightRadius: 24,
    padding: 20,
    paddingBottom: Platform.OS === 'ios' ? 36 : 20,
    maxHeight: '90%',
  },
  handle: {
    width: 40,
    height: 4,
    borderRadius: 2,
    backgroundColor: Colors.borderLight,
    alignSelf: 'center',
    marginBottom: 16,
  },
  header: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    marginBottom: 20,
  },
  title: {
    fontSize: 18,
    fontWeight: '800',
    color: Colors.textPrimary,
  },
  closeBtn: {
    fontSize: 18,
    color: Colors.textSecondary,
  },
  fieldWrapper: {
    marginBottom: 14,
  },
  fieldLabel: {
    fontSize: 11,
    fontWeight: '700',
    color: Colors.textSecondary,
    textTransform: 'uppercase',
    letterSpacing: 0.6,
    marginBottom: 8,
  },
  categoryScroll: {
    flexDirection: 'row',
  },
  categoryChip: {
    borderRadius: 20,
    paddingHorizontal: 14,
    paddingVertical: 6,
    backgroundColor: Colors.surface2,
    marginRight: 8,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  categoryChipActive: {
    backgroundColor: Colors.accent,
    borderColor: Colors.accent,
  },
  categoryChipText: {
    fontSize: 13,
    color: Colors.textSecondary,
    fontWeight: '500',
  },
  categoryChipTextActive: {
    color: Colors.textPrimary,
    fontWeight: '700',
  },
  toggleRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: Colors.surface2,
    borderRadius: 12,
    padding: 14,
    marginBottom: 20,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  toggleLabel: {
    fontSize: 14,
    fontWeight: '600',
    color: Colors.textPrimary,
    marginBottom: 2,
  },
  toggleHint: {
    fontSize: 12,
    color: Colors.textSecondary,
  },
  submitBtn: {
    backgroundColor: Colors.accent,
    borderRadius: 14,
    paddingVertical: 16,
    alignItems: 'center',
  },
  submitBtnDisabled: {
    opacity: 0.5,
  },
  submitBtnText: {
    color: Colors.textPrimary,
    fontSize: 16,
    fontWeight: '700',
  },
});
```

---

## 9.5 AdminCategoriesScreen

**Archivo:** `src/presentation/screens/admin/AdminCategoriesScreen.tsx`

Lista las categorías con nombre, slug y conteo de productos. Permite añadir, editar y eliminar mediante un Alert de acciones.

```typescript
// src/presentation/screens/admin/AdminCategoriesScreen.tsx

import React, { useEffect, useState, useCallback } from 'react';
import {
  View,
  Text,
  StyleSheet,
  FlatList,
  TouchableOpacity,
  ActivityIndicator,
  Alert,
  RefreshControl,
  TextInput,
  Modal,
  Platform,
  ListRenderItemInfo,
} from 'react-native';
import { Colors } from '../../../constants/Colors';
import { useAdminCatalogStore, AdminCategory } from '../../../domain/store/admin.catalog.store';
import AdminGuard from '../../navigation/AdminGuard';

// ─── Modal para crear/editar categoría ───────────────────────────────────────

interface CategoryModalProps {
  visible: boolean;
  category: AdminCategory | null;
  onClose: () => void;
  onSuccess: () => void;
}

function CategoryModal({
  visible,
  category,
  onClose,
  onSuccess,
}: CategoryModalProps) {
  const { createCategory, updateCategory, isSaving } = useAdminCatalogStore();
  const [name, setName] = useState('');

  useEffect(() => {
    setName(category?.name ?? '');
  }, [category, visible]);

  const handleSave = async () => {
    if (!name.trim()) {
      Alert.alert('Nombre requerido', 'Ingresa el nombre de la categoría.');
      return;
    }
    try {
      if (category) {
        await updateCategory(category.id, { name: name.trim() });
      } else {
        await createCategory({ name: name.trim() });
      }
      onSuccess();
    } catch {
      Alert.alert('Error', 'No se pudo guardar la categoría.');
    }
  };

  return (
    <Modal
      visible={visible}
      transparent
      animationType="fade"
      onRequestClose={onClose}
    >
      <View style={catModalStyles.overlay}>
        <View style={catModalStyles.dialog}>
          <Text style={catModalStyles.title}>
            {category ? 'Editar categoría' : 'Nueva categoría'}
          </Text>
          <TextInput
            style={catModalStyles.input}
            value={name}
            onChangeText={setName}
            placeholder="Nombre de la categoría"
            placeholderTextColor={Colors.textSecondary}
            autoFocus
            autoCapitalize="words"
          />
          <View style={catModalStyles.actions}>
            <TouchableOpacity style={catModalStyles.cancelBtn} onPress={onClose}>
              <Text style={catModalStyles.cancelText}>Cancelar</Text>
            </TouchableOpacity>
            <TouchableOpacity
              style={[
                catModalStyles.saveBtn,
                isSaving && catModalStyles.saveBtnDisabled,
              ]}
              onPress={handleSave}
              disabled={isSaving}
            >
              {isSaving ? (
                <ActivityIndicator color={Colors.textPrimary} size="small" />
              ) : (
                <Text style={catModalStyles.saveText}>
                  {category ? 'Guardar' : 'Crear'}
                </Text>
              )}
            </TouchableOpacity>
          </View>
        </View>
      </View>
    </Modal>
  );
}

const catModalStyles = StyleSheet.create({
  overlay: {
    flex: 1,
    backgroundColor: 'rgba(0,0,0,0.65)',
    alignItems: 'center',
    justifyContent: 'center',
    padding: 24,
  },
  dialog: {
    backgroundColor: Colors.surface,
    borderRadius: 20,
    padding: 24,
    width: '100%',
    maxWidth: 380,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  title: {
    fontSize: 17,
    fontWeight: '800',
    color: Colors.textPrimary,
    marginBottom: 16,
  },
  input: {
    backgroundColor: Colors.surface2,
    borderRadius: 10,
    borderWidth: 1,
    borderColor: Colors.borderLight,
    paddingHorizontal: 12,
    paddingVertical: 10,
    fontSize: 15,
    color: Colors.textPrimary,
    marginBottom: 20,
  },
  actions: {
    flexDirection: 'row',
    gap: 10,
  },
  cancelBtn: {
    flex: 1,
    backgroundColor: Colors.surface2,
    borderRadius: 10,
    paddingVertical: 12,
    alignItems: 'center',
    borderWidth: 1,
    borderColor: Colors.border,
  },
  cancelText: {
    color: Colors.textSecondary,
    fontWeight: '600',
  },
  saveBtn: {
    flex: 1,
    backgroundColor: Colors.accent,
    borderRadius: 10,
    paddingVertical: 12,
    alignItems: 'center',
  },
  saveBtnDisabled: {
    opacity: 0.5,
  },
  saveText: {
    color: Colors.textPrimary,
    fontWeight: '700',
  },
});

// ─── Fila de categoría ────────────────────────────────────────────────────────

function CategoryRow({
  category,
  onEdit,
  onDelete,
}: {
  category: AdminCategory;
  onEdit: () => void;
  onDelete: () => void;
}) {
  const handleLongPress = () => {
    Alert.alert(category.name, 'Selecciona una acción', [
      { text: 'Editar', onPress: onEdit },
      { text: 'Eliminar', style: 'destructive', onPress: onDelete },
      { text: 'Cancelar', style: 'cancel' },
    ]);
  };

  return (
    <TouchableOpacity
      style={catRowStyles.row}
      onPress={onEdit}
      onLongPress={handleLongPress}
      activeOpacity={0.75}
      delayLongPress={400}
    >
      <View style={catRowStyles.info}>
        <Text style={catRowStyles.name}>{category.name}</Text>
        <Text style={catRowStyles.slug}>{category.slug}</Text>
      </View>
      {category.product_count !== undefined && (
        <View style={catRowStyles.badge}>
          <Text style={catRowStyles.badgeText}>{category.product_count}</Text>
        </View>
      )}
    </TouchableOpacity>
  );
}

const catRowStyles = StyleSheet.create({
  row: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: Colors.surface,
    borderRadius: 12,
    padding: 16,
    marginBottom: 8,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  info: {
    flex: 1,
  },
  name: {
    fontSize: 15,
    fontWeight: '600',
    color: Colors.textPrimary,
    marginBottom: 2,
  },
  slug: {
    fontSize: 12,
    color: Colors.textSecondary,
    fontFamily: Platform.OS === 'ios' ? 'Menlo' : 'monospace',
  },
  badge: {
    backgroundColor: Colors.surface2,
    borderRadius: 12,
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  badgeText: {
    fontSize: 12,
    fontWeight: '700',
    color: Colors.textSecondary,
  },
});

// ─── Pantalla principal ───────────────────────────────────────────────────────

function AdminCategoriesContent() {
  const { categories, isLoading, deleteCategory, loadCategories } =
    useAdminCatalogStore();

  const [modalVisible, setModalVisible] = useState(false);
  const [editingCategory, setEditingCategory] = useState<AdminCategory | null>(
    null,
  );

  useEffect(() => {
    loadCategories();
  }, []);

  const handleRefresh = useCallback(() => {
    loadCategories();
  }, []);

  const handleDelete = (category: AdminCategory) => {
    Alert.alert(
      'Eliminar categoría',
      `¿Confirmas que deseas eliminar "${category.name}"? Los productos perderán su categoría.`,
      [
        { text: 'Cancelar', style: 'cancel' },
        {
          text: 'Eliminar',
          style: 'destructive',
          onPress: async () => {
            try {
              await deleteCategory(category.id);
            } catch {
              Alert.alert('Error', 'No se pudo eliminar la categoría.');
            }
          },
        },
      ],
    );
  };

  const renderItem = ({ item }: ListRenderItemInfo<AdminCategory>) => (
    <CategoryRow
      category={item}
      onEdit={() => {
        setEditingCategory(item);
        setModalVisible(true);
      }}
      onDelete={() => handleDelete(item)}
    />
  );

  return (
    <View style={catScreenStyles.container}>
      {isLoading && categories.length === 0 ? (
        <View style={catScreenStyles.center}>
          <ActivityIndicator size="large" color={Colors.accent} />
        </View>
      ) : (
        <FlatList
          data={categories}
          keyExtractor={(item) => String(item.id)}
          renderItem={renderItem}
          contentContainerStyle={catScreenStyles.list}
          ListHeaderComponent={
            <Text style={catScreenStyles.count}>
              {categories.length} categorías
            </Text>
          }
          ListEmptyComponent={
            <View style={catScreenStyles.empty}>
              <Text style={catScreenStyles.emptyText}>
                No hay categorías.{'\n'}Presiona + para añadir una.
              </Text>
            </View>
          }
          refreshControl={
            <RefreshControl
              refreshing={isLoading}
              onRefresh={handleRefresh}
              tintColor={Colors.accent}
            />
          }
          showsVerticalScrollIndicator={false}
        />
      )}

      {/* FAB */}
      <TouchableOpacity
        style={catScreenStyles.fab}
        onPress={() => {
          setEditingCategory(null);
          setModalVisible(true);
        }}
        activeOpacity={0.85}
      >
        <Text style={catScreenStyles.fabIcon}>+</Text>
      </TouchableOpacity>

      <CategoryModal
        visible={modalVisible}
        category={editingCategory}
        onClose={() => setModalVisible(false)}
        onSuccess={() => {
          setModalVisible(false);
          loadCategories();
        }}
      />
    </View>
  );
}

export default function AdminCategoriesScreen() {
  return (
    <AdminGuard>
      <AdminCategoriesContent />
    </AdminGuard>
  );
}

const catScreenStyles = StyleSheet.create({
  container: { flex: 1, backgroundColor: Colors.background },
  center: { flex: 1, alignItems: 'center', justifyContent: 'center' },
  list: { padding: 16, paddingBottom: 90 },
  count: {
    fontSize: 13,
    color: Colors.textSecondary,
    marginBottom: 12,
    fontWeight: '500',
  },
  empty: { alignItems: 'center', paddingTop: 60 },
  emptyText: {
    fontSize: 15,
    color: Colors.textSecondary,
    textAlign: 'center',
    lineHeight: 24,
  },
  fab: {
    position: 'absolute',
    bottom: 24,
    right: 20,
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: Colors.accent,
    alignItems: 'center',
    justifyContent: 'center',
    shadowColor: Colors.accent,
    shadowOpacity: 0.5,
    shadowRadius: 12,
    shadowOffset: { width: 0, height: 4 },
    elevation: 8,
  },
  fabIcon: {
    fontSize: 28,
    color: Colors.textPrimary,
    fontWeight: '300',
    lineHeight: 32,
  },
});
```

---

## 9.6 Integrar en Navigator

**Archivo:** `src/presentation/navigation/MainNavigator.tsx` _(actualizar)_

El tab de Admin solo se muestra cuando `authStore.user?.isStaff` es `true`. Se usa un stack interno para navegar entre `AdminProductsScreen` y `AdminCategoriesScreen`.

```typescript
// src/presentation/navigation/MainNavigator.tsx (fragmento de admin)

import { createNativeStackNavigator } from '@react-navigation/native-stack';
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';
import AdminProductsScreen from '../screens/admin/AdminProductsScreen';
import AdminCategoriesScreen from '../screens/admin/AdminCategoriesScreen';
import { useAuthStore } from '../../domain/store/auth.store';

export type AdminStackParamList = {
  AdminProducts: undefined;
  AdminCategories: undefined;
};

const AdminStack = createNativeStackNavigator<AdminStackParamList>();

function AdminNavigator() {
  // Cargar categorías al entrar al panel admin
  const { loadCategories } = useAdminCatalogStore();
  useEffect(() => {
    loadCategories();
  }, []);

  return (
    <AdminStack.Navigator
      screenOptions={{
        headerStyle: { backgroundColor: Colors.surface },
        headerTintColor: Colors.textPrimary,
        headerTitleStyle: { fontWeight: '700' },
        contentStyle: { backgroundColor: Colors.background },
      }}
    >
      <AdminStack.Screen
        name="AdminProducts"
        component={AdminProductsScreen}
        options={{
          title: 'Productos',
          headerRight: () => (
            <AdminHeaderLink label="Categorías" screen="AdminCategories" />
          ),
        }}
      />
      <AdminStack.Screen
        name="AdminCategories"
        component={AdminCategoriesScreen}
        options={{ title: 'Categorías' }}
      />
    </AdminStack.Navigator>
  );
}

// Botón en el header para navegar a Categorías
function AdminHeaderLink({
  label,
  screen,
}: {
  label: string;
  screen: keyof AdminStackParamList;
}) {
  const navigation = useNavigation<NativeStackNavigationProp<AdminStackParamList>>();
  return (
    <TouchableOpacity onPress={() => navigation.navigate(screen)}>
      <Text style={{ color: Colors.accent, fontSize: 14, fontWeight: '600' }}>
        {label}
      </Text>
    </TouchableOpacity>
  );
}

// En MainTabs: renderizar la pestaña Admin solo para usuarios staff
function MainTabs() {
  const isStaff = useAuthStore((state) => state.user?.isStaff ?? false);

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
      <Tab.Screen name="Home" component={HomeScreen} options={{ title: 'Inicio' }} />
      <Tab.Screen name="Cart" component={CartScreen} options={{ title: 'Carrito' }} />
      <Tab.Screen name="Orders" component={OrdersScreen} options={{ title: 'Pedidos' }} />
      <Tab.Screen name="ProfileTab" component={ProfileNavigator} options={{ title: 'Perfil', headerShown: false }} />

      {/* Pestaña Admin: visible solo para staff */}
      {isStaff && (
        <Tab.Screen
          name="AdminTab"
          component={AdminNavigator}
          options={{ title: 'Admin', headerShown: false }}
        />
      )}
    </Tab.Navigator>
  );
}
```

**Asegúrate de mapear `is_staff` en el AuthStore:**

```typescript
// src/domain/store/auth.store.ts
// Al recibir la respuesta de login o /api/users/me/:

interface AuthUser {
  id: number;
  username: string;
  email: string;
  isStaff: boolean;   // mapeado desde is_staff
}

// En la acción de login:
const user: AuthUser = {
  id: response.data.id,
  username: response.data.username,
  email: response.data.email,
  isStaff: response.data.is_staff,  // campo Django
};
set({ user, token: response.data.token, isAuthenticated: true });
```

---

## ✅ Checkpoint — Módulo 9

Verifica que el panel de administración funcione correctamente:

- [ ] La pestaña "Admin" en el bottom tab solo aparece cuando `authStore.user.isStaff` es `true`. Un usuario normal no la ve.
- [ ] Si un usuario sin permisos accede directamente a `AdminProductsScreen` (por ejemplo mediante deeplink), `AdminGuard` muestra la pantalla de acceso denegado en lugar del contenido.
- [ ] `AdminProductsScreen` lista todos los productos con nombre, precio, stock y toggle de `is_active`.
- [ ] Al tocar el `Switch`, el estado del producto cambia inmediatamente en la UI (actualización optimista) y se envía el `PATCH` a la API.
- [ ] El FAB (+) abre `ProductFormModal` en modo creación con todos los campos vacíos.
- [ ] Al tocar un producto, `ProductFormModal` se abre en modo edición con los datos precargados.
- [ ] Al hacer long press sobre un producto, aparece el Alert con opciones "Editar" y "Eliminar".
- [ ] Eliminar un producto muestra una confirmación antes de ejecutar el `DELETE`.
- [ ] `AdminCategoriesScreen` lista todas las categorías con nombre, slug y número de productos.
- [ ] El modal de categoría permite crear y editar con un campo de nombre simple.
- [ ] Desde `AdminProductsScreen`, el botón "Categorías" en el header navega a `AdminCategoriesScreen`.
- [ ] El selector de categorías en `ProductFormModal` muestra chips horizontales con las categorías cargadas en el store.

---

**Siguiente módulo:** El módulo 10 añade la gestión de pedidos desde el panel admin: listar todos los pedidos, filtrar por estado y actualizar el estado de un pedido mediante `POST /api/orders/{id}/update-status/`.
