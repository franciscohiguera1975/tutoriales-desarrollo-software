# ShopApp React Native — Módulo 3

## Auth — Login, Registro, AuthStore con Zustand y persistencia de sesión

---

> **Objetivo**
> Implementar el sistema completo de autenticación: tipos TypeScript, store Zustand con persistencia
> cifrada, pantallas de Login y Registro, y los navegadores que controlan el flujo de la app según
> el estado de sesión.
>
> **Checkpoint final**
> - Iniciar sesión con credenciales válidas y navegar automáticamente al área principal.
> - Registrar un nuevo usuario y quedar autenticado.
> - Cerrar y reabrir la app: la sesión se recupera de EncryptedStorage sin volver a hacer login.

---

## 3.1 Tipos TypeScript

**Archivo:** `src/domain/model/auth.types.ts`

Define las estructuras de datos que circulan por toda la feature de autenticación.

```typescript
// src/domain/model/auth.types.ts

/**
 * Datos del usuario autenticado que devuelve el backend.
 * Endpoint: POST /api/auth/login/ o POST /api/auth/register/
 */
export interface LoggedUser {
  user_id: number;
  username: string;
  email: string;
  is_staff: boolean;
}

/**
 * Tokens JWT que emite el backend al autenticarse.
 */
export interface AuthTokens {
  access: string;
  refresh: string;
}

/**
 * Estado global que maneja el AuthStore.
 * - user: null cuando no hay sesión activa.
 * - tokens: null cuando no hay sesión activa.
 * - isLoading: true mientras se ejecuta login/register/loadSession.
 * - error: mensaje de error legible para mostrar en pantalla.
 */
export interface AuthState {
  user: LoggedUser | null;
  tokens: AuthTokens | null;
  isLoading: boolean;
  error: string | null;
}

/**
 * Credenciales para el formulario de Login.
 */
export interface LoginCredentials {
  username: string;
  password: string;
}

/**
 * Datos para el formulario de Registro.
 */
export interface RegisterData {
  username: string;
  email: string;
  password: string;
  password2: string;
}
```

> **Nota:** Estos tipos se exportan y son usados por `auth.store.ts`, las pantallas y cualquier
> componente que necesite datos del usuario autenticado.

---

## 3.2 AuthStore con Zustand

**Archivo:** `src/domain/store/auth.store.ts`

El store central de autenticación. Usa `zustand` con persistencia manual en `EncryptedStorage`
(ya instalado en el módulo 1). Las funciones de red (`loginApi`, `registerApi`, `logoutApi`)
vienen de `auth.api.ts` creado en el módulo 2.

```typescript
// src/domain/store/auth.store.ts

import { create } from 'zustand';
import EncryptedStorage from 'react-native-encrypted-storage';

import { loginApi, registerApi, logoutApi } from '../../data/api/auth.api';
import type { AuthState, LoggedUser, AuthTokens, LoginCredentials, RegisterData } from '../model/auth.types';

// Clave bajo la que se guarda la sesión en EncryptedStorage
const SESSION_KEY = 'shopapp_session';

// ─── Forma completa del store (estado + acciones) ─────────────────────────────
interface AuthStore extends AuthState {
  login: (credentials: LoginCredentials) => Promise<void>;
  register: (data: RegisterData) => Promise<void>;
  logout: () => Promise<void>;
  loadSession: () => Promise<void>;
  clearError: () => void;
}

// ─── Helpers de persistencia ──────────────────────────────────────────────────

/** Guarda usuario y tokens en EncryptedStorage (cifrado AES). */
async function saveSession(user: LoggedUser, tokens: AuthTokens): Promise<void> {
  await EncryptedStorage.setItem(SESSION_KEY, JSON.stringify({ user, tokens }));
}

/** Elimina la sesión guardada. */
async function deleteSession(): Promise<void> {
  await EncryptedStorage.removeItem(SESSION_KEY);
}

/** Lee y parsea la sesión guardada. Devuelve null si no existe o hay error. */
async function readSession(): Promise<{ user: LoggedUser; tokens: AuthTokens } | null> {
  try {
    const raw = await EncryptedStorage.getItem(SESSION_KEY);
    if (!raw) return null;
    return JSON.parse(raw) as { user: LoggedUser; tokens: AuthTokens };
  } catch {
    return null;
  }
}

// ─── Store ────────────────────────────────────────────────────────────────────

export const useAuthStore = create<AuthStore>((set, get) => ({
  // ── Estado inicial ──────────────────────────────────────────────────────────
  user: null,
  tokens: null,
  isLoading: false,
  error: null,

  // ── Acciones ────────────────────────────────────────────────────────────────

  /**
   * Intenta autenticar al usuario contra el backend.
   * En caso de éxito guarda la sesión en EncryptedStorage.
   */
  login: async (credentials) => {
    set({ isLoading: true, error: null });
    try {
      const response = await loginApi(credentials);
      const user: LoggedUser = {
        user_id:  response.user_id,
        username: response.username,
        email:    response.email,
        is_staff: response.is_staff,
      };
      const tokens: AuthTokens = {
        access:  response.access,
        refresh: response.refresh,
      };
      await saveSession(user, tokens);
      set({ user, tokens, isLoading: false });
    } catch (err: any) {
      const message =
        err?.response?.data?.detail ||
        err?.response?.data?.non_field_errors?.[0] ||
        'Error al iniciar sesión. Verifica tus credenciales.';
      set({ error: message, isLoading: false });
    }
  },

  /**
   * Registra un nuevo usuario en el backend.
   * En caso de éxito deja al usuario autenticado y guarda la sesión.
   */
  register: async (data) => {
    set({ isLoading: true, error: null });
    try {
      const response = await registerApi(data);
      const user: LoggedUser = {
        user_id:  response.user_id,
        username: response.username,
        email:    response.email,
        is_staff: response.is_staff,
      };
      const tokens: AuthTokens = {
        access:  response.access,
        refresh: response.refresh,
      };
      await saveSession(user, tokens);
      set({ user, tokens, isLoading: false });
    } catch (err: any) {
      // Django REST devuelve errores por campo; los concatenamos en un mensaje
      const data = err?.response?.data;
      let message = 'Error al registrar. Revisa los datos ingresados.';
      if (data) {
        const firstField = Object.values(data)[0];
        if (Array.isArray(firstField)) message = firstField[0] as string;
      }
      set({ error: message, isLoading: false });
    }
  },

  /**
   * Cierra la sesión: llama al endpoint de logout (invalida el refresh token),
   * elimina la sesión local y limpia el store.
   */
  logout: async () => {
    const { tokens } = get();
    set({ isLoading: true });
    try {
      if (tokens?.refresh) {
        await logoutApi({ refresh: tokens.refresh });
      }
    } catch {
      // Ignorar errores de red al cerrar sesión; el token expirará solo
    } finally {
      await deleteSession();
      set({ user: null, tokens: null, isLoading: false, error: null });
    }
  },

  /**
   * Lee la sesión guardada en EncryptedStorage al arrancar la app.
   * Se llama una sola vez desde App.tsx usando useEffect.
   */
  loadSession: async () => {
    set({ isLoading: true });
    const saved = await readSession();
    if (saved) {
      set({ user: saved.user, tokens: saved.tokens, isLoading: false });
    } else {
      set({ isLoading: false });
    }
  },

  /** Limpia el mensaje de error (útil cuando el usuario edita el formulario). */
  clearError: () => set({ error: null }),
}));
```

> **Por qué no usamos `zustand/middleware` `persist` directamente?**
> El middleware `persist` de Zustand requiere un adaptador de storage compatible con su API
> asíncrona. `react-native-encrypted-storage` cifra con AES-256 y no expone la API síncrona
> que algunos adaptadores esperan. Manejamos la persistencia manualmente para tener control
> total y garantizar el cifrado.

---

## 3.3 LoginScreen

**Archivo:** `src/presentation/screens/auth/LoginScreen.tsx`

Pantalla de inicio de sesión con validación básica, indicador de carga y manejo de errores.
Usa `StyleSheet` con los colores del tema oscuro de la app.

```typescript
// src/presentation/screens/auth/LoginScreen.tsx

import React, { useState, useEffect } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ActivityIndicator,
  StyleSheet,
  KeyboardAvoidingView,
  Platform,
  ScrollView,
} from 'react-native';
import type { NativeStackScreenProps } from '@react-navigation/native-stack';

import { useAuthStore } from '../../../domain/store/auth.store';
import { Colors } from '../../../theme/colors';
import type { AuthStackParamList } from '../../navigation/AuthNavigator';

type Props = NativeStackScreenProps<AuthStackParamList, 'Login'>;

export default function LoginScreen({ navigation }: Props) {
  const [username, setUsername] = useState('');
  const [password, setPassword] = useState('');

  const { login, isLoading, error, clearError } = useAuthStore();

  // Limpia el error cuando el usuario empieza a corregir
  useEffect(() => {
    if (error) clearError();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [username, password]);

  const handleLogin = async () => {
    if (!username.trim() || !password.trim()) return;
    await login({ username: username.trim(), password });
    // Si login fue exitoso, AppNavigator detecta el cambio de estado y navega solo
  };

  return (
    <KeyboardAvoidingView
      style={styles.flex}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
    >
      <ScrollView
        contentContainerStyle={styles.container}
        keyboardShouldPersistTaps="handled"
      >
        {/* Logo / encabezado */}
        <Text style={styles.logo}>ShopApp</Text>
        <Text style={styles.subtitle}>Inicia sesión para continuar</Text>

        {/* Campos del formulario */}
        <View style={styles.formGroup}>
          <Text style={styles.label}>Usuario</Text>
          <TextInput
            style={styles.input}
            placeholder="Ingresa tu usuario"
            placeholderTextColor={Colors.textSecondary}
            value={username}
            onChangeText={setUsername}
            autoCapitalize="none"
            autoCorrect={false}
            returnKeyType="next"
          />
        </View>

        <View style={styles.formGroup}>
          <Text style={styles.label}>Contraseña</Text>
          <TextInput
            style={styles.input}
            placeholder="Ingresa tu contraseña"
            placeholderTextColor={Colors.textSecondary}
            value={password}
            onChangeText={setPassword}
            secureTextEntry
            returnKeyType="done"
            onSubmitEditing={handleLogin}
          />
        </View>

        {/* Mensaje de error */}
        {error ? (
          <View style={styles.errorBox}>
            <Text style={styles.errorText}>{error}</Text>
          </View>
        ) : null}

        {/* Botón principal */}
        <TouchableOpacity
          style={[styles.button, isLoading && styles.buttonDisabled]}
          onPress={handleLogin}
          disabled={isLoading}
          activeOpacity={0.8}
        >
          {isLoading ? (
            <ActivityIndicator color={Colors.textPrimary} />
          ) : (
            <Text style={styles.buttonText}>Iniciar sesión</Text>
          )}
        </TouchableOpacity>

        {/* Enlace a Registro */}
        <TouchableOpacity
          style={styles.linkContainer}
          onPress={() => navigation.navigate('Register')}
        >
          <Text style={styles.linkText}>
            ¿No tienes cuenta?{' '}
            <Text style={styles.linkAccent}>Regístrate</Text>
          </Text>
        </TouchableOpacity>
      </ScrollView>
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  flex: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  container: {
    flexGrow: 1,
    justifyContent: 'center',
    paddingHorizontal: 24,
    paddingVertical: 40,
  },
  logo: {
    fontSize: 36,
    fontWeight: '800',
    color: Colors.accent,
    textAlign: 'center',
    marginBottom: 8,
    letterSpacing: 1,
  },
  subtitle: {
    fontSize: 15,
    color: Colors.textSecondary,
    textAlign: 'center',
    marginBottom: 40,
  },
  formGroup: {
    marginBottom: 20,
  },
  label: {
    fontSize: 13,
    color: Colors.textSecondary,
    marginBottom: 6,
    fontWeight: '600',
    textTransform: 'uppercase',
    letterSpacing: 0.5,
  },
  input: {
    backgroundColor: Colors.surface,
    borderWidth: 1,
    borderColor: Colors.border,
    borderRadius: 10,
    paddingHorizontal: 16,
    paddingVertical: 14,
    fontSize: 16,
    color: Colors.textPrimary,
  },
  errorBox: {
    backgroundColor: '#FF525220',
    borderWidth: 1,
    borderColor: Colors.error,
    borderRadius: 8,
    padding: 12,
    marginBottom: 16,
  },
  errorText: {
    color: Colors.error,
    fontSize: 14,
  },
  button: {
    backgroundColor: Colors.accent,
    borderRadius: 10,
    paddingVertical: 16,
    alignItems: 'center',
    marginTop: 8,
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  buttonText: {
    color: Colors.textPrimary,
    fontSize: 16,
    fontWeight: '700',
  },
  linkContainer: {
    marginTop: 24,
    alignItems: 'center',
  },
  linkText: {
    color: Colors.textSecondary,
    fontSize: 14,
  },
  linkAccent: {
    color: Colors.accent,
    fontWeight: '700',
  },
});
```

---

## 3.4 RegisterScreen

**Archivo:** `src/presentation/screens/auth/RegisterScreen.tsx`

Formulario de registro con cuatro campos. Valida que las contraseñas coincidan antes de enviar.

```typescript
// src/presentation/screens/auth/RegisterScreen.tsx

import React, { useState, useEffect, useRef } from 'react';
import {
  View,
  Text,
  TextInput,
  TouchableOpacity,
  ActivityIndicator,
  StyleSheet,
  KeyboardAvoidingView,
  Platform,
  ScrollView,
  TextInput as RNTextInput,
} from 'react-native';
import type { NativeStackScreenProps } from '@react-navigation/native-stack';

import { useAuthStore } from '../../../domain/store/auth.store';
import { Colors } from '../../../theme/colors';
import type { AuthStackParamList } from '../../navigation/AuthNavigator';

type Props = NativeStackScreenProps<AuthStackParamList, 'Register'>;

export default function RegisterScreen({ navigation }: Props) {
  const [username, setUsername]   = useState('');
  const [email, setEmail]         = useState('');
  const [password, setPassword]   = useState('');
  const [password2, setPassword2] = useState('');
  const [localError, setLocalError] = useState<string | null>(null);

  // Refs para navegar entre inputs con el teclado
  const emailRef     = useRef<RNTextInput>(null);
  const passwordRef  = useRef<RNTextInput>(null);
  const password2Ref = useRef<RNTextInput>(null);

  const { register, isLoading, error, clearError } = useAuthStore();

  useEffect(() => {
    if (error) clearError();
    if (localError) setLocalError(null);
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [username, email, password, password2]);

  const handleRegister = async () => {
    // Validación local antes de llamar al backend
    if (!username.trim() || !email.trim() || !password || !password2) {
      setLocalError('Todos los campos son obligatorios.');
      return;
    }
    if (password !== password2) {
      setLocalError('Las contraseñas no coinciden.');
      return;
    }
    if (password.length < 8) {
      setLocalError('La contraseña debe tener al menos 8 caracteres.');
      return;
    }
    await register({ username: username.trim(), email: email.trim(), password, password2 });
  };

  const displayError = localError || error;

  return (
    <KeyboardAvoidingView
      style={styles.flex}
      behavior={Platform.OS === 'ios' ? 'padding' : undefined}
    >
      <ScrollView
        contentContainerStyle={styles.container}
        keyboardShouldPersistTaps="handled"
      >
        <Text style={styles.title}>Crear cuenta</Text>
        <Text style={styles.subtitle}>Únete a ShopApp y empieza a comprar</Text>

        {/* Usuario */}
        <View style={styles.formGroup}>
          <Text style={styles.label}>Usuario</Text>
          <TextInput
            style={styles.input}
            placeholder="Elige un nombre de usuario"
            placeholderTextColor={Colors.textSecondary}
            value={username}
            onChangeText={setUsername}
            autoCapitalize="none"
            autoCorrect={false}
            returnKeyType="next"
            onSubmitEditing={() => emailRef.current?.focus()}
          />
        </View>

        {/* Email */}
        <View style={styles.formGroup}>
          <Text style={styles.label}>Correo electrónico</Text>
          <TextInput
            ref={emailRef}
            style={styles.input}
            placeholder="correo@ejemplo.com"
            placeholderTextColor={Colors.textSecondary}
            value={email}
            onChangeText={setEmail}
            autoCapitalize="none"
            keyboardType="email-address"
            returnKeyType="next"
            onSubmitEditing={() => passwordRef.current?.focus()}
          />
        </View>

        {/* Contraseña */}
        <View style={styles.formGroup}>
          <Text style={styles.label}>Contraseña</Text>
          <TextInput
            ref={passwordRef}
            style={styles.input}
            placeholder="Mínimo 8 caracteres"
            placeholderTextColor={Colors.textSecondary}
            value={password}
            onChangeText={setPassword}
            secureTextEntry
            returnKeyType="next"
            onSubmitEditing={() => password2Ref.current?.focus()}
          />
        </View>

        {/* Confirmar contraseña */}
        <View style={styles.formGroup}>
          <Text style={styles.label}>Confirmar contraseña</Text>
          <TextInput
            ref={password2Ref}
            style={styles.input}
            placeholder="Repite tu contraseña"
            placeholderTextColor={Colors.textSecondary}
            value={password2}
            onChangeText={setPassword2}
            secureTextEntry
            returnKeyType="done"
            onSubmitEditing={handleRegister}
          />
        </View>

        {/* Error */}
        {displayError ? (
          <View style={styles.errorBox}>
            <Text style={styles.errorText}>{displayError}</Text>
          </View>
        ) : null}

        {/* Botón */}
        <TouchableOpacity
          style={[styles.button, isLoading && styles.buttonDisabled]}
          onPress={handleRegister}
          disabled={isLoading}
          activeOpacity={0.8}
        >
          {isLoading ? (
            <ActivityIndicator color={Colors.textPrimary} />
          ) : (
            <Text style={styles.buttonText}>Crear cuenta</Text>
          )}
        </TouchableOpacity>

        {/* Enlace a Login */}
        <TouchableOpacity
          style={styles.linkContainer}
          onPress={() => navigation.goBack()}
        >
          <Text style={styles.linkText}>
            ¿Ya tienes cuenta?{' '}
            <Text style={styles.linkAccent}>Inicia sesión</Text>
          </Text>
        </TouchableOpacity>
      </ScrollView>
    </KeyboardAvoidingView>
  );
}

const styles = StyleSheet.create({
  flex: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  container: {
    flexGrow: 1,
    justifyContent: 'center',
    paddingHorizontal: 24,
    paddingVertical: 40,
  },
  title: {
    fontSize: 30,
    fontWeight: '800',
    color: Colors.textPrimary,
    textAlign: 'center',
    marginBottom: 8,
  },
  subtitle: {
    fontSize: 14,
    color: Colors.textSecondary,
    textAlign: 'center',
    marginBottom: 36,
  },
  formGroup: {
    marginBottom: 18,
  },
  label: {
    fontSize: 13,
    color: Colors.textSecondary,
    marginBottom: 6,
    fontWeight: '600',
    textTransform: 'uppercase',
    letterSpacing: 0.5,
  },
  input: {
    backgroundColor: Colors.surface,
    borderWidth: 1,
    borderColor: Colors.border,
    borderRadius: 10,
    paddingHorizontal: 16,
    paddingVertical: 14,
    fontSize: 16,
    color: Colors.textPrimary,
  },
  errorBox: {
    backgroundColor: '#FF525220',
    borderWidth: 1,
    borderColor: Colors.error,
    borderRadius: 8,
    padding: 12,
    marginBottom: 16,
  },
  errorText: {
    color: Colors.error,
    fontSize: 14,
  },
  button: {
    backgroundColor: Colors.accent,
    borderRadius: 10,
    paddingVertical: 16,
    alignItems: 'center',
    marginTop: 8,
  },
  buttonDisabled: {
    opacity: 0.6,
  },
  buttonText: {
    color: Colors.textPrimary,
    fontSize: 16,
    fontWeight: '700',
  },
  linkContainer: {
    marginTop: 24,
    alignItems: 'center',
  },
  linkText: {
    color: Colors.textSecondary,
    fontSize: 14,
  },
  linkAccent: {
    color: Colors.accent,
    fontWeight: '700',
  },
});
```

---

## 3.5 AuthNavigator

**Archivo:** `src/presentation/navigation/AuthNavigator.tsx`

Stack navigator que agrupa las pantallas públicas (Login y Register). Define el tipo de los
parámetros de ruta exportado para que los componentes puedan tiparlo correctamente.

```typescript
// src/presentation/navigation/AuthNavigator.tsx

import React from 'react';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

import LoginScreen    from '../screens/auth/LoginScreen';
import RegisterScreen from '../screens/auth/RegisterScreen';
import { Colors } from '../../theme/colors';

// Tipo de los parámetros de cada ruta de este stack
export type AuthStackParamList = {
  Login:    undefined;
  Register: undefined;
};

const Stack = createNativeStackNavigator<AuthStackParamList>();

export default function AuthNavigator() {
  return (
    <Stack.Navigator
      screenOptions={{
        headerShown: false,            // Las pantallas de auth manejan su propio encabezado
        contentStyle: { backgroundColor: Colors.background },
        animation: 'slide_from_right',
      }}
    >
      <Stack.Screen name="Login"    component={LoginScreen}    />
      <Stack.Screen name="Register" component={RegisterScreen} />
    </Stack.Navigator>
  );
}
```

---

## 3.6 AppNavigator

**Archivo:** `src/presentation/navigation/AppNavigator.tsx`

Componente raíz de navegación. Observa el estado de `useAuthStore` y decide si mostrar el
flujo de autenticación o el flujo principal. Muestra un `ActivityIndicator` mientras
`loadSession` verifica si hay sesión guardada.

```typescript
// src/presentation/navigation/AppNavigator.tsx

import React from 'react';
import { View, ActivityIndicator, StyleSheet } from 'react-native';

import { useAuthStore } from '../../domain/store/auth.store';
import AuthNavigator    from './AuthNavigator';
import MainNavigator    from './MainNavigator';
import { Colors } from '../../theme/colors';

export default function AppNavigator() {
  const { user, isLoading } = useAuthStore();

  // Pantalla de splash mientras se recupera la sesión cifrada
  if (isLoading) {
    return (
      <View style={styles.splash}>
        <ActivityIndicator size="large" color={Colors.accent} />
      </View>
    );
  }

  // Decisión: ¿hay usuario autenticado?
  return user ? <MainNavigator /> : <AuthNavigator />;
}

const styles = StyleSheet.create({
  splash: {
    flex: 1,
    backgroundColor: Colors.background,
    justifyContent: 'center',
    alignItems: 'center',
  },
});
```

> **MainNavigator (placeholder):** Crea el archivo `src/presentation/navigation/MainNavigator.tsx` con un
> bottom tab navigator básico. Se completará en el módulo 4 al agregar el catálogo.

```typescript
// src/presentation/navigation/MainNavigator.tsx  ← PLACEHOLDER para este módulo

import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { useAuthStore } from '../../domain/store/auth.store';
import { Colors } from '../../theme/colors';

export default function MainNavigator() {
  const { user, logout } = useAuthStore();

  return (
    <View style={styles.container}>
      <Text style={styles.welcome}>
        Bienvenido, {user?.username} 👋
      </Text>
      <TouchableOpacity style={styles.button} onPress={logout}>
        <Text style={styles.buttonText}>Cerrar sesión</Text>
      </TouchableOpacity>
    </View>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
    justifyContent: 'center',
    alignItems: 'center',
    gap: 20,
  },
  welcome: {
    color: Colors.textPrimary,
    fontSize: 20,
    fontWeight: '700',
  },
  button: {
    backgroundColor: Colors.error,
    paddingHorizontal: 24,
    paddingVertical: 12,
    borderRadius: 8,
  },
  buttonText: {
    color: Colors.textPrimary,
    fontWeight: '700',
  },
});
```

---

## 3.7 Actualizar App.tsx

**Archivo:** `App.tsx` (raíz del proyecto)

Envuelve toda la app con los proveedores necesarios y ejecuta `loadSession` al montar.

```typescript
// App.tsx

import React, { useEffect } from 'react';
import { StyleSheet } from 'react-native';
import { NavigationContainer } from '@react-navigation/native';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { SafeAreaProvider } from 'react-native-safe-area-context';

import AppNavigator from './src/presentation/navigation/AppNavigator';
import { useAuthStore } from './src/domain/store/auth.store';
import { Colors } from './src/theme/colors';

export default function App() {
  const loadSession = useAuthStore((state) => state.loadSession);

  // Recuperar sesión persistida al arrancar la app
  useEffect(() => {
    loadSession();
  }, [loadSession]);

  return (
    <GestureHandlerRootView style={styles.root}>
      <SafeAreaProvider>
        <NavigationContainer>
          <AppNavigator />
        </NavigationContainer>
      </SafeAreaProvider>
    </GestureHandlerRootView>
  );
}

const styles = StyleSheet.create({
  root: {
    flex: 1,
    backgroundColor: Colors.background,
  },
});
```

> **Orden de los proveedores (importante):**
> 1. `GestureHandlerRootView` — debe ser el contenedor más externo (requerido por
>    `react-native-gesture-handler` que usa React Navigation).
> 2. `SafeAreaProvider` — gestiona los insets de notch y barra de navegación del SO.
> 3. `NavigationContainer` — contexto de React Navigation.

---

## Resumen de archivos del Módulo 3

| Archivo | Descripción |
|---|---|
| `src/domain/model/auth.types.ts` | Interfaces TypeScript del dominio de autenticación |
| `src/domain/store/auth.store.ts` | Store Zustand con persistencia cifrada |
| `src/presentation/screens/auth/LoginScreen.tsx` | Pantalla de inicio de sesión |
| `src/presentation/screens/auth/RegisterScreen.tsx` | Pantalla de registro de usuario |
| `src/presentation/navigation/AuthNavigator.tsx` | Stack navigator de pantallas públicas |
| `src/presentation/navigation/AppNavigator.tsx` | Navegador raíz con lógica de sesión |
| `src/presentation/navigation/MainNavigator.tsx` | Placeholder del área principal (módulo 4) |
| `App.tsx` | Punto de entrada con proveedores y carga de sesión |

---

## Checkpoint final — Módulo 3

**Prueba 1: Login exitoso**
1. Ejecuta `npx react-native run-android` (o `run-ios`).
2. Ingresa un usuario y contraseña válidos del backend Django.
3. La app debe navegar automáticamente a la pantalla de bienvenida.
4. Cierra la app completamente (no solo minimizar).
5. Reabre la app: debe mostrar directamente la pantalla de bienvenida sin pedir login.

**Prueba 2: Logout**
1. Presiona "Cerrar sesión".
2. La app debe mostrar el LoginScreen.
3. Cierra y reabre: debe mostrar LoginScreen (sesión eliminada).

**Prueba 3: Registro**
1. En LoginScreen, presiona "Regístrate".
2. Completa el formulario con datos nuevos.
3. Al registrar, la app navega al área principal y persiste la sesión.

**Prueba 4: Credenciales incorrectas**
1. Ingresa un usuario o contraseña incorrectos.
2. Debe aparecer el cuadro de error con el mensaje del backend.
