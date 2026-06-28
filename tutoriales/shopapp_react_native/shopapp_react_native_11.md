# ShopApp React Native — Módulo 11

## Panel Admin — Gestión de Usuarios

**Duración estimada:** 3–4 horas
**Prerequisitos:** Módulo 10 completado. Usuario con `is_staff = true`.

---

> **Objetivo**
> Implementar la gestión completa de usuarios desde el panel de administración: listar, buscar, crear, editar, activar/desactivar y confirmar eliminación de usuarios.

---

> **Checkpoint final**
> Al terminar este módulo deberás poder:
> - Listar todos los usuarios con búsqueda por username
> - Filtrar por categoría: Todos / Staff / Clientes / Activos / Inactivos
> - Crear un nuevo usuario staff desde un formulario modal
> - Activar o desactivar un usuario con actualización optimista de la UI
> - Ver confirmación de alerta antes de eliminar un usuario

---

## 11.1 AdminUsersStore

**Archivo:** `src/domain/store/admin.users.store.ts`

### Tipos

Crea `src/domain/model/admin.users.types.ts`:

```typescript
export interface User {
  id: number;
  username: string;
  email: string;
  first_name: string;
  last_name: string;
  is_staff: boolean;
  is_active: boolean;
  date_joined: string;
  num_orders: number;
  avatar_url: string | null;
}

export type UserFilter = 'all' | 'staff' | 'clients' | 'active' | 'inactive';

export interface CreateUserPayload {
  username: string;
  email: string;
  first_name: string;
  last_name: string;
  password: string;
  is_staff: boolean;
}

export interface UpdateUserPayload {
  username?: string;
  email?: string;
  first_name?: string;
  last_name?: string;
  is_staff?: boolean;
}

export interface AdminUsersState {
  users: User[];
  isLoading: boolean;
  isSubmitting: boolean;
  error: string | null;
  searchQuery: string;
  filterChip: UserFilter;
}

export interface AdminUsersActions {
  loadUsers: () => Promise<void>;
  createUser: (payload: CreateUserPayload) => Promise<void>;
  updateUser: (id: number, payload: UpdateUserPayload) => Promise<void>;
  deleteUser: (id: number) => Promise<void>;
  toggleActive: (id: number) => Promise<void>;
  setSearchQuery: (query: string) => void;
  setFilterChip: (filter: UserFilter) => void;
  resetError: () => void;
}
```

### Store con CRUD y toggleActive optimista

```typescript
// src/domain/store/admin.users.store.ts
import { create } from 'zustand';
import {
  AdminUsersState,
  AdminUsersActions,
  User,
  CreateUserPayload,
  UpdateUserPayload,
  UserFilter,
} from '../model/admin.users.types';
import { apiClient } from '../../data/api/api.client';

const initialState: AdminUsersState = {
  users: [],
  isLoading: false,
  isSubmitting: false,
  error: null,
  searchQuery: '',
  filterChip: 'all',
};

export const useAdminUsersStore = create<AdminUsersState & AdminUsersActions>(
  (set, get) => ({
    ...initialState,

    loadUsers: async () => {
      const { searchQuery, filterChip } = get();
      set({ isLoading: true, error: null });

      try {
        const params: Record<string, string> = {};

        if (searchQuery.trim()) {
          params.search = searchQuery.trim();
        }

        // Mapear chips de filtro a parámetros del backend
        if (filterChip === 'staff') params.is_staff = 'true';
        if (filterChip === 'clients') params.is_staff = 'false';
        if (filterChip === 'active') params.is_active = 'true';
        if (filterChip === 'inactive') params.is_active = 'false';

        const response = await apiClient.get<User[]>('/api/users/', { params });
        set({ users: response.data, isLoading: false });
      } catch (error: any) {
        set({
          error: error?.response?.data?.detail || 'Error al cargar usuarios',
          isLoading: false,
        });
      }
    },

    createUser: async (payload: CreateUserPayload) => {
      set({ isSubmitting: true, error: null });
      try {
        const response = await apiClient.post<User>('/api/users/', payload);
        set(state => ({
          users: [response.data, ...state.users],
          isSubmitting: false,
        }));
      } catch (error: any) {
        const errorData = error?.response?.data;
        // Django retorna errores de campo como objeto; convertir a string
        const errorMessage =
          typeof errorData === 'object'
            ? Object.entries(errorData)
                .map(([k, v]) => `${k}: ${v}`)
                .join('\n')
            : 'Error al crear el usuario';
        set({ error: errorMessage, isSubmitting: false });
        throw error;
      }
    },

    updateUser: async (id: number, payload: UpdateUserPayload) => {
      set({ isSubmitting: true, error: null });
      try {
        const response = await apiClient.patch<User>(
          `/api/users/${id}/`,
          payload,
        );
        set(state => ({
          users: state.users.map(u => (u.id === id ? response.data : u)),
          isSubmitting: false,
        }));
      } catch (error: any) {
        set({
          error: error?.response?.data?.detail || 'Error al actualizar',
          isSubmitting: false,
        });
        throw error;
      }
    },

    deleteUser: async (id: number) => {
      set({ isSubmitting: true, error: null });
      try {
        await apiClient.delete(`/api/users/${id}/`);
        set(state => ({
          users: state.users.filter(u => u.id !== id),
          isSubmitting: false,
        }));
      } catch (error: any) {
        set({
          error: error?.response?.data?.detail || 'Error al eliminar',
          isSubmitting: false,
        });
        throw error;
      }
    },

    // toggleActive con actualización optimista
    toggleActive: async (id: number) => {
      const user = get().users.find(u => u.id === id);
      if (!user) return;

      // 1. Actualización optimista: cambiar is_active localmente antes de la llamada
      const newIsActive = !user.is_active;
      set(state => ({
        users: state.users.map(u =>
          u.id === id ? { ...u, is_active: newIsActive } : u,
        ),
      }));

      try {
        // 2. Llamada al backend
        const response = await apiClient.post<User>(
          `/api/users/${id}/toggle-active/`,
        );

        // 3. Confirmar con la respuesta real del servidor
        set(state => ({
          users: state.users.map(u => (u.id === id ? response.data : u)),
        }));
      } catch (error: any) {
        // 4. Revertir si hay error
        set(state => ({
          users: state.users.map(u =>
            u.id === id ? { ...u, is_active: user.is_active } : u,
          ),
          error: 'No se pudo cambiar el estado del usuario',
        }));
      }
    },

    setSearchQuery: (query: string) => {
      set({ searchQuery: query });
    },

    setFilterChip: (filter: UserFilter) => {
      set({ filterChip: filter });
      get().loadUsers();
    },

    resetError: () => set({ error: null }),
  }),
);
```

---

## 11.2 AdminUsersScreen

**Archivo:** `src/presentation/screens/admin/AdminUsersScreen.tsx`

Lista buscable de usuarios con debounce de 400 ms para evitar llamadas excesivas a la API mientras el usuario escribe.

```typescript
import React, { useEffect, useCallback, useRef, useState } from 'react';
import {
  View,
  FlatList,
  Text,
  TextInput,
  TouchableOpacity,
  StyleSheet,
  ActivityIndicator,
  RefreshControl,
} from 'react-native';
import { NativeStackScreenProps } from '@react-navigation/native-stack';
import { AdminStackParamList } from '../../../presentation/navigation/admin.navigator';
import { useAdminUsersStore } from '../../../domain/store/admin.users.store';
import { User, UserFilter } from '../../../domain/model/admin.users.types';
import { Colors } from '../../../theme/colors';
import { UserListItem } from './components/UserListItem';
import { UserFormModal } from './components/UserFormModal';

type Props = NativeStackScreenProps<AdminStackParamList, 'AdminUsers'>;

const FILTER_CHIPS: Array<{ label: string; value: UserFilter }> = [
  { label: 'Todos', value: 'all' },
  { label: 'Staff', value: 'staff' },
  { label: 'Clientes', value: 'clients' },
  { label: 'Activos', value: 'active' },
  { label: 'Inactivos', value: 'inactive' },
];

export const AdminUsersScreen: React.FC<Props> = () => {
  const [formModalVisible, setFormModalVisible] = useState(false);
  const [editingUser, setEditingUser] = useState<User | null>(null);

  const {
    users,
    isLoading,
    searchQuery,
    filterChip,
    loadUsers,
    setSearchQuery,
    setFilterChip,
    toggleActive,
    deleteUser,
  } = useAdminUsersStore();

  // Referencia al timeout para el debounce de búsqueda
  const debounceRef = useRef<ReturnType<typeof setTimeout> | null>(null);

  useEffect(() => {
    loadUsers();
  }, []);

  const handleSearchChange = useCallback(
    (text: string) => {
      setSearchQuery(text);

      if (debounceRef.current) {
        clearTimeout(debounceRef.current);
      }

      debounceRef.current = setTimeout(() => {
        loadUsers();
      }, 400);
    },
    [setSearchQuery, loadUsers],
  );

  const handleEdit = useCallback((user: User) => {
    setEditingUser(user);
    setFormModalVisible(true);
  }, []);

  const handleCloseModal = useCallback(() => {
    setFormModalVisible(false);
    setEditingUser(null);
  }, []);

  const renderItem = useCallback(
    ({ item }: { item: User }) => (
      <UserListItem
        user={item}
        onEdit={() => handleEdit(item)}
        onToggleActive={() => toggleActive(item.id)}
        onDelete={() => deleteUser(item.id)}
      />
    ),
    [handleEdit, toggleActive, deleteUser],
  );

  return (
    <View style={styles.container}>
      {/* Barra de búsqueda */}
      <View style={styles.searchContainer}>
        <TextInput
          style={styles.searchInput}
          placeholder="Buscar por username..."
          placeholderTextColor={Colors.textSecondary}
          value={searchQuery}
          onChangeText={handleSearchChange}
          autoCapitalize="none"
          autoCorrect={false}
        />
      </View>

      {/* Chips de filtro */}
      <View style={styles.chipsRow}>
        <FlatList
          horizontal
          data={FILTER_CHIPS}
          keyExtractor={item => item.value}
          showsHorizontalScrollIndicator={false}
          contentContainerStyle={styles.chipsList}
          renderItem={({ item }) => {
            const isActive = filterChip === item.value;
            return (
              <TouchableOpacity
                style={[styles.chip, isActive && styles.chipActive]}
                onPress={() => setFilterChip(item.value)}>
                <Text
                  style={[
                    styles.chipText,
                    isActive && styles.chipTextActive,
                  ]}>
                  {item.label}
                </Text>
              </TouchableOpacity>
            );
          }}
        />
      </View>

      {/* Lista de usuarios */}
      <FlatList
        data={users}
        keyExtractor={item => item.id.toString()}
        renderItem={renderItem}
        contentContainerStyle={styles.listContent}
        refreshControl={
          <RefreshControl
            refreshing={isLoading}
            onRefresh={loadUsers}
            tintColor={Colors.accent}
          />
        }
        ListEmptyComponent={
          !isLoading ? (
            <View style={styles.emptyContainer}>
              <Text style={styles.emptyText}>
                {searchQuery ? 'Sin resultados para la búsqueda' : 'No hay usuarios'}
              </Text>
            </View>
          ) : (
            <ActivityIndicator
              size="large"
              color={Colors.accent}
              style={{ marginTop: 40 }}
            />
          )
        }
      />

      {/* Botón flotante para crear usuario */}
      <TouchableOpacity
        style={styles.fab}
        onPress={() => {
          setEditingUser(null);
          setFormModalVisible(true);
        }}>
        <Text style={styles.fabText}>+</Text>
      </TouchableOpacity>

      <UserFormModal
        visible={formModalVisible}
        editingUser={editingUser}
        onClose={handleCloseModal}
        onSuccess={loadUsers}
      />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  searchContainer: {
    padding: 12,
    backgroundColor: Colors.surface,
    borderBottomWidth: 1,
    borderBottomColor: Colors.border,
  },
  searchInput: {
    backgroundColor: Colors.surface2,
    borderRadius: 10,
    paddingHorizontal: 14,
    paddingVertical: 10,
    color: Colors.textPrimary,
    fontSize: 14,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  chipsRow: {
    backgroundColor: Colors.surface,
    paddingVertical: 8,
    borderBottomWidth: 1,
    borderBottomColor: Colors.border,
  },
  chipsList: {
    paddingHorizontal: 12,
    gap: 8,
  },
  chip: {
    paddingHorizontal: 14,
    paddingVertical: 6,
    borderRadius: 20,
    borderWidth: 1,
    borderColor: Colors.border,
    backgroundColor: Colors.surface2,
  },
  chipActive: {
    backgroundColor: Colors.accent,
    borderColor: Colors.accent,
  },
  chipText: {
    color: Colors.textSecondary,
    fontSize: 13,
    fontWeight: '500',
  },
  chipTextActive: {
    color: Colors.textPrimary,
  },
  listContent: {
    padding: 16,
    gap: 10,
    paddingBottom: 80,
  },
  emptyContainer: {
    alignItems: 'center',
    paddingTop: 60,
  },
  emptyText: {
    color: Colors.textSecondary,
    fontSize: 15,
  },
  fab: {
    position: 'absolute',
    bottom: 24,
    right: 24,
    width: 56,
    height: 56,
    borderRadius: 28,
    backgroundColor: Colors.accent,
    alignItems: 'center',
    justifyContent: 'center',
    elevation: 6,
    shadowColor: Colors.accent,
    shadowOffset: { width: 0, height: 4 },
    shadowOpacity: 0.4,
    shadowRadius: 8,
  },
  fabText: {
    color: Colors.textPrimary,
    fontSize: 28,
    fontWeight: '300',
    lineHeight: 30,
  },
});
```

### Componente UserListItem

Crea `src/presentation/screens/admin/components/UserListItem.tsx`:

```typescript
import React from 'react';
import {
  View,
  Text,
  TouchableOpacity,
  StyleSheet,
  Alert,
  Switch,
} from 'react-native';
import { User } from '../../../../domain/model/admin.users.types';
import { Colors } from '../../../../theme/colors';

interface Props {
  user: User;
  onEdit: () => void;
  onToggleActive: () => void;
  onDelete: () => void;
}

export const UserListItem: React.FC<Props> = ({
  user,
  onEdit,
  onToggleActive,
  onDelete,
}) => {
  const initials = `${user.first_name.charAt(0)}${user.last_name.charAt(0)}`.toUpperCase()
    || user.username.charAt(0).toUpperCase();

  const handleDeletePress = () => {
    Alert.alert(
      'Eliminar usuario',
      `¿Estás seguro de que deseas eliminar a "${user.username}"? Esta acción no se puede deshacer.`,
      [
        { text: 'Cancelar', style: 'cancel' },
        {
          text: 'Eliminar',
          style: 'destructive',
          onPress: onDelete,
        },
      ],
    );
  };

  return (
    <View style={[styles.card, !user.is_active && styles.cardInactive]}>
      <View style={styles.avatarCircle}>
        <Text style={styles.initials}>{initials}</Text>
      </View>

      <View style={styles.info}>
        <View style={styles.nameRow}>
          <Text style={styles.username}>{user.username}</Text>
          {user.is_staff && (
            <View style={styles.staffBadge}>
              <Text style={styles.staffText}>Staff</Text>
            </View>
          )}
        </View>
        <Text style={styles.email}>{user.email}</Text>
        <Text style={styles.meta}>
          {user.num_orders} pedido{user.num_orders !== 1 ? 's' : ''}
          {'  •  '}
          Desde {new Date(user.date_joined).toLocaleDateString('es-EC')}
        </Text>
      </View>

      <View style={styles.actions}>
        {/* Toggle activo/inactivo */}
        <Switch
          value={user.is_active}
          onValueChange={onToggleActive}
          trackColor={{ false: Colors.border, true: Colors.accent + '80' }}
          thumbColor={user.is_active ? Colors.accent : Colors.textSecondary}
        />

        {/* Botón editar */}
        <TouchableOpacity style={styles.editButton} onPress={onEdit}>
          <Text style={styles.editText}>Editar</Text>
        </TouchableOpacity>

        {/* Botón eliminar */}
        <TouchableOpacity style={styles.deleteButton} onPress={handleDeletePress}>
          <Text style={styles.deleteText}>✕</Text>
        </TouchableOpacity>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  card: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: Colors.surface,
    borderRadius: 12,
    padding: 12,
    borderWidth: 1,
    borderColor: Colors.border,
    gap: 12,
  },
  cardInactive: {
    opacity: 0.6,
  },
  avatarCircle: {
    width: 44,
    height: 44,
    borderRadius: 22,
    backgroundColor: Colors.accent + '33',
    alignItems: 'center',
    justifyContent: 'center',
  },
  initials: {
    color: Colors.accent,
    fontSize: 16,
    fontWeight: '700',
  },
  info: {
    flex: 1,
    gap: 2,
  },
  nameRow: {
    flexDirection: 'row',
    alignItems: 'center',
    gap: 8,
  },
  username: {
    color: Colors.textPrimary,
    fontSize: 14,
    fontWeight: '600',
  },
  staffBadge: {
    backgroundColor: Colors.accent + '22',
    paddingHorizontal: 8,
    paddingVertical: 2,
    borderRadius: 10,
  },
  staffText: {
    color: Colors.accent,
    fontSize: 11,
    fontWeight: '600',
  },
  email: {
    color: Colors.textSecondary,
    fontSize: 12,
  },
  meta: {
    color: Colors.textSecondary,
    fontSize: 11,
    marginTop: 2,
  },
  actions: {
    alignItems: 'center',
    gap: 6,
  },
  editButton: {
    paddingHorizontal: 10,
    paddingVertical: 4,
    borderRadius: 8,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  editText: {
    color: Colors.textSecondary,
    fontSize: 12,
  },
  deleteButton: {
    width: 28,
    height: 28,
    borderRadius: 14,
    backgroundColor: Colors.error + '22',
    alignItems: 'center',
    justifyContent: 'center',
  },
  deleteText: {
    color: Colors.error,
    fontSize: 12,
    fontWeight: '700',
  },
});
```

---

## 11.3 UserFormModal

**Archivo:** `src/presentation/screens/admin/components/UserFormModal.tsx`

Formulario modal para crear o editar usuarios. En modo creación el campo password es requerido; en edición es opcional.

```typescript
import React, { useState, useEffect } from 'react';
import {
  Modal,
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ScrollView,
  Switch,
  StyleSheet,
  ActivityIndicator,
  Alert,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import { User, CreateUserPayload, UpdateUserPayload } from '../../../../domain/model/admin.users.types';
import { useAdminUsersStore } from '../../../../domain/store/admin.users.store';
import { Colors } from '../../../../theme/colors';

interface Props {
  visible: boolean;
  editingUser: User | null;
  onClose: () => void;
  onSuccess: () => void;
}

interface FormState {
  username: string;
  email: string;
  first_name: string;
  last_name: string;
  password: string;
  is_staff: boolean;
}

const initialForm: FormState = {
  username: '',
  email: '',
  first_name: '',
  last_name: '',
  password: '',
  is_staff: false,
};

export const UserFormModal: React.FC<Props> = ({
  visible,
  editingUser,
  onClose,
  onSuccess,
}) => {
  const isEditing = editingUser !== null;
  const [form, setForm] = useState<FormState>(initialForm);
  const [formErrors, setFormErrors] = useState<Partial<FormState>>({});

  const { createUser, updateUser, isSubmitting, error, resetError } =
    useAdminUsersStore();

  // Inicializar el formulario con datos del usuario al editar
  useEffect(() => {
    if (editingUser) {
      setForm({
        username: editingUser.username,
        email: editingUser.email,
        first_name: editingUser.first_name,
        last_name: editingUser.last_name,
        password: '',
        is_staff: editingUser.is_staff,
      });
    } else {
      setForm(initialForm);
    }
    setFormErrors({});
    resetError();
  }, [editingUser, visible]);

  const validate = (): boolean => {
    const errors: Partial<FormState> = {};

    if (!form.username.trim()) errors.username = 'El username es requerido';
    if (!form.email.trim()) errors.email = 'El email es requerido';
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(form.email)) {
      errors.email = 'Email inválido';
    }
    if (!isEditing && !form.password) {
      errors.password = 'La contraseña es requerida al crear';
    }
    if (form.password && form.password.length < 8) {
      errors.password = 'Mínimo 8 caracteres';
    }

    setFormErrors(errors);
    return Object.keys(errors).length === 0;
  };

  const handleSubmit = async () => {
    if (!validate()) return;

    try {
      if (isEditing && editingUser) {
        const payload: UpdateUserPayload = {
          username: form.username,
          email: form.email,
          first_name: form.first_name,
          last_name: form.last_name,
          is_staff: form.is_staff,
        };
        await updateUser(editingUser.id, payload);
      } else {
        const payload: CreateUserPayload = {
          username: form.username,
          email: form.email,
          first_name: form.first_name,
          last_name: form.last_name,
          password: form.password,
          is_staff: form.is_staff,
        };
        await createUser(payload);
      }

      onSuccess();
      onClose();
    } catch {
      // El error ya está en el store; mostrar alerta si es necesario
      Alert.alert(
        'Error',
        error || 'Ocurrió un error. Revisa los campos e intenta de nuevo.',
      );
    }
  };

  const setField = (field: keyof FormState, value: string | boolean) => {
    setForm(prev => ({ ...prev, [field]: value }));
    if (formErrors[field]) {
      setFormErrors(prev => ({ ...prev, [field]: undefined }));
    }
  };

  return (
    <Modal
      visible={visible}
      transparent
      animationType="slide"
      onRequestClose={onClose}>
      <KeyboardAvoidingView
        style={{ flex: 1 }}
        behavior={Platform.OS === 'ios' ? 'padding' : undefined}>
        <TouchableOpacity
          style={styles.overlay}
          activeOpacity={1}
          onPress={onClose}
        />
        <View style={styles.sheet}>
          <View style={styles.handle} />
          <Text style={styles.title}>
            {isEditing ? 'Editar Usuario' : 'Nuevo Usuario'}
          </Text>

          <ScrollView showsVerticalScrollIndicator={false}>
            <View style={styles.form}>
              <FormField
                label="Username *"
                value={form.username}
                onChangeText={v => setField('username', v)}
                error={formErrors.username}
                autoCapitalize="none"
              />
              <FormField
                label="Email *"
                value={form.email}
                onChangeText={v => setField('email', v)}
                error={formErrors.email}
                keyboardType="email-address"
                autoCapitalize="none"
              />
              <FormField
                label="Nombre"
                value={form.first_name}
                onChangeText={v => setField('first_name', v)}
              />
              <FormField
                label="Apellido"
                value={form.last_name}
                onChangeText={v => setField('last_name', v)}
              />
              <FormField
                label={isEditing ? 'Nueva contraseña (opcional)' : 'Contraseña *'}
                value={form.password}
                onChangeText={v => setField('password', v)}
                error={formErrors.password}
                secureTextEntry
                autoCapitalize="none"
              />

              {/* Toggle is_staff */}
              <View style={styles.toggleRow}>
                <View>
                  <Text style={styles.toggleLabel}>Es administrador (staff)</Text>
                  <Text style={styles.toggleDesc}>
                    Tendrá acceso al panel de administración
                  </Text>
                </View>
                <Switch
                  value={form.is_staff}
                  onValueChange={v => setField('is_staff', v)}
                  trackColor={{
                    false: Colors.border,
                    true: Colors.accent + '80',
                  }}
                  thumbColor={form.is_staff ? Colors.accent : Colors.textSecondary}
                />
              </View>
            </View>
          </ScrollView>

          <View style={styles.actions}>
            <TouchableOpacity
              style={styles.cancelButton}
              onPress={onClose}
              disabled={isSubmitting}>
              <Text style={styles.cancelText}>Cancelar</Text>
            </TouchableOpacity>
            <TouchableOpacity
              style={[
                styles.submitButton,
                isSubmitting && styles.submitButtonDisabled,
              ]}
              onPress={handleSubmit}
              disabled={isSubmitting}>
              {isSubmitting ? (
                <ActivityIndicator size="small" color={Colors.textPrimary} />
              ) : (
                <Text style={styles.submitText}>
                  {isEditing ? 'Guardar cambios' : 'Crear usuario'}
                </Text>
              )}
            </TouchableOpacity>
          </View>
        </View>
      </KeyboardAvoidingView>
    </Modal>
  );
};

interface FormFieldProps {
  label: string;
  value: string;
  onChangeText: (v: string) => void;
  error?: string;
  secureTextEntry?: boolean;
  keyboardType?: 'default' | 'email-address';
  autoCapitalize?: 'none' | 'sentences' | 'words';
}

const FormField: React.FC<FormFieldProps> = ({
  label,
  value,
  onChangeText,
  error,
  secureTextEntry,
  keyboardType = 'default',
  autoCapitalize = 'sentences',
}) => (
  <View style={fieldStyles.container}>
    <Text style={fieldStyles.label}>{label}</Text>
    <TextInput
      style={[fieldStyles.input, error ? fieldStyles.inputError : null]}
      value={value}
      onChangeText={onChangeText}
      placeholderTextColor={Colors.textSecondary}
      secureTextEntry={secureTextEntry}
      keyboardType={keyboardType}
      autoCapitalize={autoCapitalize}
    />
    {error ? <Text style={fieldStyles.errorText}>{error}</Text> : null}
  </View>
);

const fieldStyles = StyleSheet.create({
  container: { gap: 4 },
  label: { color: Colors.textSecondary, fontSize: 13, fontWeight: '500' },
  input: {
    backgroundColor: Colors.surface2,
    borderRadius: 10,
    paddingHorizontal: 14,
    paddingVertical: 11,
    color: Colors.textPrimary,
    fontSize: 14,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  inputError: { borderColor: Colors.error },
  errorText: { color: Colors.error, fontSize: 12 },
});

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
    maxHeight: '90%',
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
    marginBottom: 20,
  },
  form: {
    gap: 16,
    paddingBottom: 8,
  },
  toggleRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    backgroundColor: Colors.surface2,
    borderRadius: 10,
    padding: 14,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  toggleLabel: {
    color: Colors.textPrimary,
    fontSize: 14,
    fontWeight: '500',
  },
  toggleDesc: {
    color: Colors.textSecondary,
    fontSize: 12,
    marginTop: 2,
  },
  actions: {
    flexDirection: 'row',
    gap: 12,
    marginTop: 20,
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
  submitButton: {
    flex: 2,
    paddingVertical: 14,
    borderRadius: 10,
    backgroundColor: Colors.accent,
    alignItems: 'center',
  },
  submitButtonDisabled: { opacity: 0.6 },
  submitText: {
    color: Colors.textPrimary,
    fontSize: 15,
    fontWeight: '700',
  },
});
```

---

## 11.4 Toggle active optimista

La actualización optimista ya está incluida en `useAdminUsersStore.toggleActive` (sección 11.1), pero aquí se explica el flujo en detalle:

```
1. Usuario presiona el Switch del UserListItem
2. onToggleActive() llama a store.toggleActive(user.id)
3. El store actualiza is_active localmente ANTES de la llamada HTTP
   → La UI refleja el cambio instantáneamente (sin esperar la red)
4. Se envía POST /api/users/{id}/toggle-active/ al backend
5a. Si tiene éxito: se confirma con la respuesta real del servidor
5b. Si hay error: se revierte is_active al valor original y se muestra el error
```

**Por qué es importante:** En conexiones lentas (3G o WiFi débil), sin optimistic UI el usuario presiona el switch y no pasa nada visualmente hasta que llega la respuesta (500–2000 ms). Con la actualización optimista la respuesta es inmediata y el usuario tiene mejor percepción de la app.

---

## 11.5 Confirmación de acciones destructivas

La eliminación de un usuario es irreversible, por lo que siempre se muestra un `Alert.alert` de confirmación antes de ejecutar la llamada al backend. Este patrón ya está implementado en `UserListItem.handleDeletePress`.

```typescript
// Patrón estándar para acciones destructivas en la app
const confirmarEliminacion = (
  nombre: string,
  onConfirm: () => void,
) => {
  Alert.alert(
    'Confirmar eliminación',
    `¿Eliminar "${nombre}"? Esta acción no se puede deshacer.`,
    [
      { text: 'Cancelar', style: 'cancel' },
      { text: 'Eliminar', style: 'destructive', onPress: onConfirm },
    ],
    { cancelable: true }, // Android: cerrar tocando fuera del alert
  );
};
```

Aplica este mismo patrón en cualquier otra acción destructiva de la app: eliminar productos, cancelar pedidos, etc.

---

> **Checkpoint — Módulo 11**
>
> Verifica que puedes:
> - [ ] Ver la lista de usuarios con avatar de iniciales, badge de staff e indicador activo/inactivo
> - [ ] Buscar por username con debounce (la búsqueda no ejecuta una llamada en cada tecla)
> - [ ] Filtrar por Staff, Clientes, Activos, Inactivos con los chips
> - [ ] Crear un nuevo usuario staff con el formulario modal y ver validación en campos requeridos
> - [ ] Editar un usuario existente (el campo contraseña queda opcional)
> - [ ] Activar/desactivar un usuario con el Switch y ver el cambio inmediato en la UI (optimistic update)
> - [ ] Ver el Alert de confirmación antes de eliminar y cancelar sin borrar

---

**Siguiente módulo:** Módulo 12 — Imágenes: Fotos de producto y avatar de usuario
