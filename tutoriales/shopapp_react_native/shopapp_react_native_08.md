# ShopApp React Native — Módulo 8

## Perfil de Usuario — Ver, editar y cambiar contraseña

**Objetivo:** Implementar la sección de perfil de usuario que permite visualizar los datos de la cuenta, editar nombre y email, cambiar la contraseña y cerrar sesión. Los datos se obtienen desde `GET /api/users/profile/` y se actualizan con `PATCH /api/users/profile/`.

> **Checkpoint final:** El usuario puede ver su avatar, nombre y email en `ProfileScreen`, editar sus datos desde `EditProfileScreen`, cambiar su contraseña desde `ChangePasswordScreen` y cerrar sesión volviendo a la pantalla de login.

---

## 8.1 Tipos de perfil

**Archivo:** `src/domain/model/profile.types.ts`

Define las interfaces para los datos del perfil y los payloads de actualización.

```typescript
// src/domain/model/profile.types.ts

export interface UserProfile {
  id: number;
  username: string;
  email: string;
  first_name: string;
  last_name: string;
  avatar_url: string | null;
}

// Campos que se pueden editar mediante PATCH /api/users/profile/
export interface UpdateProfilePayload {
  first_name?: string;
  last_name?: string;
  email?: string;
  // avatar se envía como FormData multipart cuando se incluye
}

// Payload para cambiar contraseña
export interface ChangePasswordPayload {
  current_password: string;
  new_password: string;
  new_password2: string;
}

// Respuesta de error genérica de DRF
export interface DRFError {
  detail?: string;
  [field: string]: string | string[] | undefined;
}

// Extrae el primer mensaje de error de la respuesta DRF
export function extractDRFError(err: any): string {
  const data: DRFError = err?.response?.data ?? {};
  if (data.detail) return data.detail;
  // Busca el primer campo con error
  for (const key of Object.keys(data)) {
    const val = data[key];
    if (typeof val === 'string') return `${key}: ${val}`;
    if (Array.isArray(val) && val.length > 0) return `${key}: ${val[0]}`;
  }
  return 'Ocurrió un error inesperado';
}
```

---

## 8.2 ProfileStore

**Archivo:** `src/domain/store/profile.store.ts`

Store de Zustand con estado del perfil y tres acciones principales: cargar, actualizar y cambiar contraseña.

```typescript
// src/domain/store/profile.store.ts

import { create } from 'zustand';
import {
  UserProfile,
  UpdateProfilePayload,
  ChangePasswordPayload,
  extractDRFError,
} from '../model/profile.types';
import { apiClient } from '../../data/api/apiClient';

interface ProfileState {
  // Estado
  profile: UserProfile | null;
  isLoading: boolean;
  isSaving: boolean;
  error: string | null;

  // Acciones
  loadProfile: () => Promise<void>;
  updateProfile: (data: UpdateProfilePayload) => Promise<void>;
  changePassword: (
    currentPassword: string,
    newPassword: string,
    newPassword2: string,
  ) => Promise<void>;
  clearError: () => void;
  resetProfile: () => void;
}

export const useProfileStore = create<ProfileState>((set) => ({
  profile: null,
  isLoading: false,
  isSaving: false,
  error: null,

  // Carga el perfil del usuario autenticado
  loadProfile: async () => {
    set({ isLoading: true, error: null });
    try {
      const response = await apiClient.get<UserProfile>('/api/users/profile/');
      set({ profile: response.data, isLoading: false });
    } catch (err: any) {
      set({
        error: extractDRFError(err),
        isLoading: false,
      });
    }
  },

  // Actualiza los campos del perfil mediante PATCH
  // Si data contiene un archivo (avatar), construye un FormData multipart
  updateProfile: async (data: UpdateProfilePayload) => {
    set({ isSaving: true, error: null });
    try {
      const response = await apiClient.patch<UserProfile>(
        '/api/users/profile/',
        data,
      );
      set({ profile: response.data, isSaving: false });
    } catch (err: any) {
      set({
        error: extractDRFError(err),
        isSaving: false,
      });
      throw err; // Re-lanza para que el caller pueda manejar el error localmente
    }
  },

  // Envía el cambio de contraseña
  changePassword: async (
    currentPassword: string,
    newPassword: string,
    newPassword2: string,
  ) => {
    set({ isSaving: true, error: null });
    try {
      await apiClient.post('/api/auth/change-password/', {
        current_password: currentPassword,
        new_password: newPassword,
        new_password2: newPassword2,
      } as ChangePasswordPayload);
      set({ isSaving: false });
    } catch (err: any) {
      set({
        error: extractDRFError(err),
        isSaving: false,
      });
      throw err;
    }
  },

  clearError: () => set({ error: null }),
  resetProfile: () =>
    set({ profile: null, isLoading: false, isSaving: false, error: null }),
}));
```

**Puntos clave:**
- `updateProfile` y `changePassword` relanza el error después de actualizarlo en el store para que las pantallas puedan mostrarlo localmente con `try/catch`.
- `isSaving` se usa para deshabilitar el botón de guardar mientras se espera la respuesta, evitando doble envío.
- `resetProfile` debe llamarse al hacer logout (en `AuthStore` o en el handler de logout).

---

## 8.3 ProfileScreen

**Archivo:** `src/presentation/screens/profile/ProfileScreen.tsx`

Pantalla principal del perfil. Muestra avatar circular, nombre completo, username y email. Incluye botones para editar, cambiar contraseña y cerrar sesión.

```typescript
// src/presentation/screens/profile/ProfileScreen.tsx

import React, { useEffect } from 'react';
import {
  View,
  Text,
  StyleSheet,
  Image,
  TouchableOpacity,
  ActivityIndicator,
  ScrollView,
  Alert,
} from 'react-native';
import { useNavigation } from '@react-navigation/native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { Colors } from '../../../constants/Colors';
import { useProfileStore } from '../../../domain/store/profile.store';
import { useAuthStore } from '../../../domain/store/auth.store';
import { ProfileStackParamList } from '../../../presentation/navigation/types';

type ProfileNav = NativeStackNavigationProp<ProfileStackParamList>;

function AvatarDisplay({
  uri,
  initials,
}: {
  uri: string | null;
  initials: string;
}) {
  if (uri) {
    return <Image source={{ uri }} style={avatarStyles.image} />;
  }
  return (
    <View style={avatarStyles.placeholder}>
      <Text style={avatarStyles.initials}>{initials}</Text>
    </View>
  );
}

const avatarStyles = StyleSheet.create({
  image: {
    width: 90,
    height: 90,
    borderRadius: 45,
    borderWidth: 3,
    borderColor: Colors.accent,
  },
  placeholder: {
    width: 90,
    height: 90,
    borderRadius: 45,
    backgroundColor: Colors.surface2,
    borderWidth: 3,
    borderColor: Colors.accent,
    alignItems: 'center',
    justifyContent: 'center',
  },
  initials: {
    fontSize: 32,
    fontWeight: '700',
    color: Colors.accent,
  },
});

function MenuRow({
  label,
  onPress,
  destructive,
}: {
  label: string;
  onPress: () => void;
  destructive?: boolean;
}) {
  return (
    <TouchableOpacity style={menuStyles.row} onPress={onPress} activeOpacity={0.7}>
      <Text style={[menuStyles.label, destructive && menuStyles.destructive]}>
        {label}
      </Text>
      <Text style={menuStyles.arrow}>›</Text>
    </TouchableOpacity>
  );
}

const menuStyles = StyleSheet.create({
  row: {
    flexDirection: 'row',
    alignItems: 'center',
    justifyContent: 'space-between',
    paddingVertical: 14,
    borderBottomWidth: 1,
    borderBottomColor: Colors.border,
  },
  label: {
    fontSize: 15,
    color: Colors.textPrimary,
    fontWeight: '500',
  },
  destructive: {
    color: Colors.error,
  },
  arrow: {
    fontSize: 20,
    color: Colors.textSecondary,
  },
});

export default function ProfileScreen() {
  const navigation = useNavigation<ProfileNav>();
  const { profile, isLoading, error, loadProfile } = useProfileStore();
  const { logout } = useAuthStore();

  useEffect(() => {
    loadProfile();
  }, []);

  const handleLogout = () => {
    Alert.alert(
      'Cerrar sesión',
      '¿Seguro que quieres cerrar sesión?',
      [
        { text: 'Cancelar', style: 'cancel' },
        {
          text: 'Cerrar sesión',
          style: 'destructive',
          onPress: () => {
            logout();
            // El AuthStore debe actualizar el estado de autenticación
            // y el navigator raíz redirige automáticamente a AuthStack
          },
        },
      ],
    );
  };

  // Genera iniciales a partir del nombre o username
  const getInitials = (): string => {
    if (!profile) return '?';
    const first = profile.first_name.charAt(0).toUpperCase();
    const last = profile.last_name.charAt(0).toUpperCase();
    return first && last ? `${first}${last}` : profile.username.slice(0, 2).toUpperCase();
  };

  if (isLoading && !profile) {
    return (
      <View style={styles.center}>
        <ActivityIndicator size="large" color={Colors.accent} />
      </View>
    );
  }

  if (error && !profile) {
    return (
      <View style={styles.center}>
        <Text style={styles.errorText}>{error}</Text>
        <TouchableOpacity style={styles.retryBtn} onPress={loadProfile}>
          <Text style={styles.retryText}>Reintentar</Text>
        </TouchableOpacity>
      </View>
    );
  }

  return (
    <ScrollView
      style={styles.container}
      contentContainerStyle={styles.scroll}
      showsVerticalScrollIndicator={false}
    >
      {/* Sección de avatar y datos principales */}
      <View style={styles.avatarSection}>
        <AvatarDisplay uri={profile?.avatar_url ?? null} initials={getInitials()} />
        <Text style={styles.fullName}>
          {profile?.first_name} {profile?.last_name}
        </Text>
        <Text style={styles.username}>@{profile?.username}</Text>
        <Text style={styles.email}>{profile?.email}</Text>
      </View>

      {/* Menú de opciones */}
      <View style={styles.menuCard}>
        <MenuRow
          label="Editar perfil"
          onPress={() => navigation.navigate('EditProfile')}
        />
        <MenuRow
          label="Cambiar contraseña"
          onPress={() => navigation.navigate('ChangePassword')}
        />
        <MenuRow
          label="Cerrar sesión"
          onPress={handleLogout}
          destructive
        />
      </View>

      {/* ID de usuario (información secundaria) */}
      <Text style={styles.userId}>ID de cuenta: {profile?.id}</Text>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  scroll: {
    padding: 20,
    paddingBottom: 40,
  },
  center: {
    flex: 1,
    alignItems: 'center',
    justifyContent: 'center',
    backgroundColor: Colors.background,
  },
  avatarSection: {
    alignItems: 'center',
    paddingVertical: 28,
  },
  fullName: {
    fontSize: 22,
    fontWeight: '700',
    color: Colors.textPrimary,
    marginTop: 16,
    marginBottom: 4,
  },
  username: {
    fontSize: 14,
    color: Colors.accent,
    marginBottom: 6,
  },
  email: {
    fontSize: 14,
    color: Colors.textSecondary,
  },
  menuCard: {
    backgroundColor: Colors.surface,
    borderRadius: 16,
    paddingHorizontal: 16,
    marginBottom: 24,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  userId: {
    fontSize: 12,
    color: Colors.textSecondary,
    textAlign: 'center',
  },
  errorText: {
    color: Colors.error,
    fontSize: 15,
    textAlign: 'center',
    marginBottom: 16,
    paddingHorizontal: 24,
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

---

## 8.4 EditProfileScreen

**Archivo:** `src/presentation/screens/profile/EditProfileScreen.tsx`

Formulario para editar `first_name`, `last_name` y `email`. Inicializa los campos con los datos actuales del store y llama a `profileStore.updateProfile()` al guardar.

```typescript
// src/presentation/screens/profile/EditProfileScreen.tsx

import React, { useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TextInput,
  TouchableOpacity,
  ScrollView,
  ActivityIndicator,
  Alert,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import { useNavigation } from '@react-navigation/native';
import { Colors } from '../../../constants/Colors';
import { useProfileStore } from '../../../domain/store/profile.store';

function FormField({
  label,
  value,
  onChangeText,
  placeholder,
  keyboardType,
  autoCapitalize,
}: {
  label: string;
  value: string;
  onChangeText: (text: string) => void;
  placeholder?: string;
  keyboardType?: 'default' | 'email-address';
  autoCapitalize?: 'none' | 'words' | 'sentences';
}) {
  return (
    <View style={fieldStyles.container}>
      <Text style={fieldStyles.label}>{label}</Text>
      <TextInput
        style={fieldStyles.input}
        value={value}
        onChangeText={onChangeText}
        placeholder={placeholder ?? label}
        placeholderTextColor={Colors.textSecondary}
        keyboardType={keyboardType ?? 'default'}
        autoCapitalize={autoCapitalize ?? 'sentences'}
        autoCorrect={false}
      />
    </View>
  );
}

const fieldStyles = StyleSheet.create({
  container: {
    marginBottom: 16,
  },
  label: {
    fontSize: 12,
    fontWeight: '600',
    color: Colors.textSecondary,
    textTransform: 'uppercase',
    letterSpacing: 0.6,
    marginBottom: 6,
  },
  input: {
    backgroundColor: Colors.surface,
    borderRadius: 12,
    borderWidth: 1,
    borderColor: Colors.border,
    paddingHorizontal: 14,
    paddingVertical: 12,
    fontSize: 15,
    color: Colors.textPrimary,
  },
});

export default function EditProfileScreen() {
  const navigation = useNavigation();
  const { profile, isSaving, updateProfile } = useProfileStore();

  const [firstName, setFirstName] = useState(profile?.first_name ?? '');
  const [lastName, setLastName] = useState(profile?.last_name ?? '');
  const [email, setEmail] = useState(profile?.email ?? '');

  // Detecta si hubo cambios para habilitar el botón de guardar
  const hasChanges =
    firstName !== (profile?.first_name ?? '') ||
    lastName !== (profile?.last_name ?? '') ||
    email !== (profile?.email ?? '');

  const handleSave = async () => {
    if (!hasChanges) return;

    // Validación mínima de email
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) {
      Alert.alert('Email inválido', 'Ingresa un correo electrónico válido.');
      return;
    }

    try {
      await updateProfile({
        first_name: firstName.trim(),
        last_name: lastName.trim(),
        email: email.trim(),
      });
      Alert.alert('Perfil actualizado', 'Tus datos han sido guardados.', [
        { text: 'OK', onPress: () => navigation.goBack() },
      ]);
    } catch {
      // El store ya guarda el error; mostramos un Alert adicional
      Alert.alert(
        'Error al guardar',
        'No se pudieron guardar los cambios. Intenta de nuevo.',
      );
    }
  };

  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
    >
      <ScrollView
        style={styles.container}
        contentContainerStyle={styles.scroll}
        keyboardShouldPersistTaps="handled"
        showsVerticalScrollIndicator={false}
      >
        <Text style={styles.title}>Editar perfil</Text>
        <Text style={styles.subtitle}>
          El nombre de usuario no es editable.
        </Text>

        {/* Campo no editable: username */}
        <View style={styles.readonlyField}>
          <Text style={styles.readonlyLabel}>Nombre de usuario</Text>
          <Text style={styles.readonlyValue}>@{profile?.username}</Text>
        </View>

        <FormField
          label="Nombre"
          value={firstName}
          onChangeText={setFirstName}
          placeholder="Tu nombre"
          autoCapitalize="words"
        />
        <FormField
          label="Apellido"
          value={lastName}
          onChangeText={setLastName}
          placeholder="Tu apellido"
          autoCapitalize="words"
        />
        <FormField
          label="Correo electrónico"
          value={email}
          onChangeText={setEmail}
          placeholder="correo@ejemplo.com"
          keyboardType="email-address"
          autoCapitalize="none"
        />

        <TouchableOpacity
          style={[
            styles.saveBtn,
            (!hasChanges || isSaving) && styles.saveBtnDisabled,
          ]}
          onPress={handleSave}
          disabled={!hasChanges || isSaving}
          activeOpacity={0.8}
        >
          {isSaving ? (
            <ActivityIndicator color={Colors.textPrimary} />
          ) : (
            <Text style={styles.saveBtnText}>Guardar cambios</Text>
          )}
        </TouchableOpacity>
      </ScrollView>
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  scroll: {
    padding: 20,
    paddingBottom: 40,
  },
  title: {
    fontSize: 22,
    fontWeight: '800',
    color: Colors.textPrimary,
    marginBottom: 4,
  },
  subtitle: {
    fontSize: 13,
    color: Colors.textSecondary,
    marginBottom: 24,
  },
  readonlyField: {
    backgroundColor: Colors.surface2,
    borderRadius: 12,
    paddingHorizontal: 14,
    paddingVertical: 12,
    marginBottom: 16,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  readonlyLabel: {
    fontSize: 11,
    color: Colors.textSecondary,
    textTransform: 'uppercase',
    letterSpacing: 0.6,
    marginBottom: 2,
  },
  readonlyValue: {
    fontSize: 15,
    color: Colors.textSecondary,
  },
  saveBtn: {
    backgroundColor: Colors.accent,
    borderRadius: 14,
    paddingVertical: 16,
    alignItems: 'center',
    marginTop: 8,
  },
  saveBtnDisabled: {
    opacity: 0.5,
  },
  saveBtnText: {
    color: Colors.textPrimary,
    fontSize: 16,
    fontWeight: '700',
  },
});
```

**Puntos clave:**
- `hasChanges` compara el estado actual del formulario con los valores del store para habilitar el botón solo cuando hay modificaciones.
- `KeyboardAvoidingView` garantiza que el formulario suba cuando aparece el teclado en iOS.
- Tras guardar con éxito, el Alert muestra confirmación y llama a `navigation.goBack()` para volver al perfil.

---

## 8.5 ChangePasswordScreen

**Archivo:** `src/presentation/screens/profile/ChangePasswordScreen.tsx`

Formulario con tres campos: contraseña actual, nueva contraseña y confirmación. Llama a `profileStore.changePassword()` y valida que las contraseñas nuevas coincidan antes de enviar.

```typescript
// src/presentation/screens/profile/ChangePasswordScreen.tsx

import React, { useState } from 'react';
import {
  View,
  Text,
  StyleSheet,
  TextInput,
  TouchableOpacity,
  ScrollView,
  ActivityIndicator,
  Alert,
  KeyboardAvoidingView,
  Platform,
} from 'react-native';
import { useNavigation } from '@react-navigation/native';
import { Colors } from '../../../constants/Colors';
import { useProfileStore } from '../../../domain/store/profile.store';

interface PasswordFieldProps {
  label: string;
  value: string;
  onChangeText: (text: string) => void;
  placeholder?: string;
  errorMsg?: string;
}

function PasswordField({
  label,
  value,
  onChangeText,
  placeholder,
  errorMsg,
}: PasswordFieldProps) {
  const [visible, setVisible] = useState(false);

  return (
    <View style={fieldStyles.container}>
      <Text style={fieldStyles.label}>{label}</Text>
      <View
        style={[fieldStyles.inputWrapper, errorMsg ? fieldStyles.inputError : null]}
      >
        <TextInput
          style={fieldStyles.input}
          value={value}
          onChangeText={onChangeText}
          placeholder={placeholder ?? label}
          placeholderTextColor={Colors.textSecondary}
          secureTextEntry={!visible}
          autoCapitalize="none"
          autoCorrect={false}
        />
        <TouchableOpacity
          onPress={() => setVisible((v) => !v)}
          style={fieldStyles.eyeBtn}
          hitSlop={{ top: 8, bottom: 8, left: 8, right: 8 }}
        >
          <Text style={fieldStyles.eyeIcon}>{visible ? '🙈' : '👁'}</Text>
        </TouchableOpacity>
      </View>
      {errorMsg ? (
        <Text style={fieldStyles.errorText}>{errorMsg}</Text>
      ) : null}
    </View>
  );
}

const fieldStyles = StyleSheet.create({
  container: {
    marginBottom: 16,
  },
  label: {
    fontSize: 12,
    fontWeight: '600',
    color: Colors.textSecondary,
    textTransform: 'uppercase',
    letterSpacing: 0.6,
    marginBottom: 6,
  },
  inputWrapper: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: Colors.surface,
    borderRadius: 12,
    borderWidth: 1,
    borderColor: Colors.border,
    paddingHorizontal: 14,
  },
  inputError: {
    borderColor: Colors.error,
  },
  input: {
    flex: 1,
    paddingVertical: 12,
    fontSize: 15,
    color: Colors.textPrimary,
  },
  eyeBtn: {
    paddingLeft: 8,
  },
  eyeIcon: {
    fontSize: 16,
  },
  errorText: {
    fontSize: 12,
    color: Colors.error,
    marginTop: 4,
  },
});

interface FormErrors {
  currentPassword?: string;
  newPassword?: string;
  newPassword2?: string;
}

function validateForm(
  current: string,
  newPwd: string,
  newPwd2: string,
): FormErrors {
  const errors: FormErrors = {};
  if (!current) errors.currentPassword = 'Ingresa tu contraseña actual';
  if (newPwd.length < 8)
    errors.newPassword = 'La contraseña debe tener al menos 8 caracteres';
  if (newPwd !== newPwd2)
    errors.newPassword2 = 'Las contraseñas nuevas no coinciden';
  return errors;
}

export default function ChangePasswordScreen() {
  const navigation = useNavigation();
  const { isSaving, changePassword } = useProfileStore();

  const [currentPassword, setCurrentPassword] = useState('');
  const [newPassword, setNewPassword] = useState('');
  const [newPassword2, setNewPassword2] = useState('');
  const [errors, setErrors] = useState<FormErrors>({});

  const handleSubmit = async () => {
    // Validar en cliente antes de enviar
    const formErrors = validateForm(currentPassword, newPassword, newPassword2);
    if (Object.keys(formErrors).length > 0) {
      setErrors(formErrors);
      return;
    }
    setErrors({});

    try {
      await changePassword(currentPassword, newPassword, newPassword2);
      Alert.alert(
        'Contraseña actualizada',
        'Tu contraseña ha sido cambiada con éxito.',
        [{ text: 'OK', onPress: () => navigation.goBack() }],
      );
    } catch (err: any) {
      // El error de servidor puede indicar contraseña actual incorrecta
      const serverMsg =
        err?.response?.data?.current_password?.[0] ??
        err?.response?.data?.detail ??
        'Error al cambiar la contraseña. Verifica tus datos.';
      Alert.alert('Error', serverMsg);
    }
  };

  return (
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
    >
      <ScrollView
        style={styles.container}
        contentContainerStyle={styles.scroll}
        keyboardShouldPersistTaps="handled"
        showsVerticalScrollIndicator={false}
      >
        <Text style={styles.title}>Cambiar contraseña</Text>
        <Text style={styles.subtitle}>
          Elige una contraseña segura con al menos 8 caracteres.
        </Text>

        <PasswordField
          label="Contraseña actual"
          value={currentPassword}
          onChangeText={(t) => {
            setCurrentPassword(t);
            setErrors((e) => ({ ...e, currentPassword: undefined }));
          }}
          placeholder="Tu contraseña actual"
          errorMsg={errors.currentPassword}
        />

        <PasswordField
          label="Nueva contraseña"
          value={newPassword}
          onChangeText={(t) => {
            setNewPassword(t);
            setErrors((e) => ({ ...e, newPassword: undefined }));
          }}
          placeholder="Mínimo 8 caracteres"
          errorMsg={errors.newPassword}
        />

        <PasswordField
          label="Confirmar nueva contraseña"
          value={newPassword2}
          onChangeText={(t) => {
            setNewPassword2(t);
            setErrors((e) => ({ ...e, newPassword2: undefined }));
          }}
          placeholder="Repite la nueva contraseña"
          errorMsg={errors.newPassword2}
        />

        {/* Indicador de fortaleza de contraseña */}
        {newPassword.length > 0 && (
          <PasswordStrengthBar password={newPassword} />
        )}

        <TouchableOpacity
          style={[styles.submitBtn, isSaving && styles.submitBtnDisabled]}
          onPress={handleSubmit}
          disabled={isSaving}
          activeOpacity={0.8}
        >
          {isSaving ? (
            <ActivityIndicator color={Colors.textPrimary} />
          ) : (
            <Text style={styles.submitBtnText}>Cambiar contraseña</Text>
          )}
        </TouchableOpacity>
      </ScrollView>
    </KeyboardAvoidingView>
  );
}

// Componente auxiliar: barra de fortaleza de contraseña
function PasswordStrengthBar({ password }: { password: string }) {
  const strength = calculateStrength(password);
  const colors = ['#FF5252', '#FFA000', '#4CAF50'];
  const labels = ['Débil', 'Regular', 'Fuerte'];

  return (
    <View style={strengthStyles.container}>
      <View style={strengthStyles.bars}>
        {[0, 1, 2].map((i) => (
          <View
            key={i}
            style={[
              strengthStyles.bar,
              i <= strength && { backgroundColor: colors[strength] },
            ]}
          />
        ))}
      </View>
      <Text style={[strengthStyles.label, { color: colors[strength] }]}>
        {labels[strength]}
      </Text>
    </View>
  );
}

function calculateStrength(pwd: string): number {
  let score = 0;
  if (pwd.length >= 8) score++;
  if (/[A-Z]/.test(pwd) && /[0-9]/.test(pwd)) score++;
  if (/[^A-Za-z0-9]/.test(pwd)) score++;
  return Math.min(score, 2) as 0 | 1 | 2;
}

const strengthStyles = StyleSheet.create({
  container: {
    flexDirection: 'row',
    alignItems: 'center',
    marginBottom: 20,
    gap: 10,
  },
  bars: {
    flex: 1,
    flexDirection: 'row',
    gap: 4,
  },
  bar: {
    flex: 1,
    height: 4,
    borderRadius: 2,
    backgroundColor: Colors.surface2,
  },
  label: {
    fontSize: 12,
    fontWeight: '600',
    minWidth: 50,
  },
});

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  scroll: {
    padding: 20,
    paddingBottom: 40,
  },
  title: {
    fontSize: 22,
    fontWeight: '800',
    color: Colors.textPrimary,
    marginBottom: 4,
  },
  subtitle: {
    fontSize: 13,
    color: Colors.textSecondary,
    marginBottom: 24,
    lineHeight: 20,
  },
  submitBtn: {
    backgroundColor: Colors.accent,
    borderRadius: 14,
    paddingVertical: 16,
    alignItems: 'center',
    marginTop: 8,
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

## 8.6 Integrar en MainNavigator

**Archivo:** `src/presentation/navigation/MainNavigator.tsx` _(actualizar)_

Crea un stack de navegación específico para el perfil y añade la pestaña en el bottom tab navigator.

```typescript
// src/presentation/navigation/MainNavigator.tsx (fragmento de perfil)

import { createNativeStackNavigator } from '@react-navigation/native-stack';
import ProfileScreen from '../screens/profile/ProfileScreen';
import EditProfileScreen from '../screens/profile/EditProfileScreen';
import ChangePasswordScreen from '../screens/profile/ChangePasswordScreen';

// Tipos del stack de perfil
export type ProfileStackParamList = {
  Profile: undefined;
  EditProfile: undefined;
  ChangePassword: undefined;
};

const ProfileStack = createNativeStackNavigator<ProfileStackParamList>();

function ProfileNavigator() {
  return (
    <ProfileStack.Navigator
      screenOptions={{
        headerStyle: { backgroundColor: Colors.surface },
        headerTintColor: Colors.textPrimary,
        headerTitleStyle: { fontWeight: '700' },
        contentStyle: { backgroundColor: Colors.background },
      }}
    >
      <ProfileStack.Screen
        name="Profile"
        component={ProfileScreen}
        options={{ title: 'Mi Perfil' }}
      />
      <ProfileStack.Screen
        name="EditProfile"
        component={EditProfileScreen}
        options={{ title: 'Editar Perfil' }}
      />
      <ProfileStack.Screen
        name="ChangePassword"
        component={ChangePasswordScreen}
        options={{ title: 'Cambiar Contraseña' }}
      />
    </ProfileStack.Navigator>
  );
}

// Dentro de MainTabs (Bottom Tab Navigator) — añadir pestaña Profile:
// <Tab.Screen
//   name="ProfileTab"
//   component={ProfileNavigator}
//   options={{ title: 'Perfil', headerShown: false }}
// />
```

**Estructura de navegación del módulo:**

```
MainTabs
  └── Tab: Perfil
        └── ProfileNavigator (Stack)
              ├── ProfileScreen         (raíz)
              ├── EditProfileScreen     (push)
              └── ChangePasswordScreen  (push)
```

**Logout y limpieza de estado:**

```typescript
// En src/domain/store/auth.store.ts — acción logout:
import { useProfileStore } from './profile.store';
import { useOrdersStore } from './orders.store';

logout: () => {
  // Limpiar token almacenado
  AsyncStorage.removeItem('auth_token');
  // Limpiar stores de datos
  useProfileStore.getState().resetProfile();
  useOrdersStore.getState().resetStore();
  // Actualizar estado de auth para que el navigator redirija a AuthStack
  set({ user: null, token: null, isAuthenticated: false });
},
```

---

## ✅ Checkpoint — Módulo 8

Verifica que la gestión de perfil funcione de extremo a extremo:

- [ ] La pestaña "Perfil" aparece en el bottom tab navigator.
- [ ] `ProfileScreen` carga los datos del usuario desde `GET /api/users/profile/` y muestra avatar circular (o iniciales si `avatar_url` es `null`), nombre completo, `@username` y email.
- [ ] El botón "Editar perfil" navega a `EditProfileScreen` con los campos precargados con los valores actuales.
- [ ] El botón "Guardar cambios" queda deshabilitado si no hay modificaciones en el formulario.
- [ ] Al guardar, se llama a `PATCH /api/users/profile/` y los datos actualizados se reflejan en `ProfileScreen` al volver.
- [ ] `ChangePasswordScreen` valida en cliente: contraseña actual requerida, nueva contraseña mínimo 8 caracteres, confirmación debe coincidir.
- [ ] La barra de fortaleza de contraseña cambia de rojo a naranja a verde según la complejidad.
- [ ] Al cambiar la contraseña con éxito, se muestra un Alert de confirmación y se vuelve a `ProfileScreen`.
- [ ] El botón "Cerrar sesión" muestra un Alert de confirmación antes de ejecutar el logout.
- [ ] Tras el logout, el estado de todos los stores se limpia y el navigator redirige a la pantalla de autenticación.

---

**Siguiente módulo:** El módulo 9 implementa el panel de administración para gestionar el catálogo de productos y categorías, accesible únicamente para usuarios con `is_staff: true`.
