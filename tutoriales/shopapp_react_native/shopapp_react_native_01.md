# ShopApp React Native — Módulo 1
## Setup + Estructura + Design System + Configuración base

> **Objetivo:** Crear el proyecto React Native con TypeScript, instalar todas las dependencias de la serie, configurar NativeWind, establecer la estructura de carpetas con Clean Architecture, definir el Design System de colores, configurar la navegación base y verificar que la app levanta correctamente en el emulador.
> **Checkpoint final:** La app muestra una pantalla de bienvenida con el logo ShopApp usando los colores del Design System, la navegación base funciona y la consola no reporta errores de compilación.

---

## 1.1 Prerequisitos

Antes de crear el proyecto verifica que tienes instaladas las herramientas necesarias. Esta serie usa React Native CLI (no Expo) para tener acceso completo a módulos nativos como `react-native-encrypted-storage`.

### Node.js 18+

```bash
node --version
# Debe ser >= 18.0.0
```

Si necesitas múltiples versiones de Node usa `nvm`:

```bash
nvm install 18
nvm use 18
```

### Java 17 (requerido para Android)

React Native 0.73+ requiere Java 17. Verifica con:

```bash
java -version
# openjdk version "17.x.x"
```

En Ubuntu/Debian:

```bash
sudo apt install openjdk-17-jdk
```

En macOS con Homebrew:

```bash
brew install openjdk@17
echo 'export PATH="/opt/homebrew/opt/openjdk@17/bin:$PATH"' >> ~/.zshrc
```

### Android Studio y Android SDK

1. Descarga Android Studio desde https://developer.android.com/studio
2. Durante la instalación selecciona **Android SDK**, **Android SDK Platform** y **Android Virtual Device**
3. Abre **SDK Manager** (Tools → SDK Manager) e instala:
   - Android SDK Platform 34 (o la más reciente)
   - Android SDK Build-Tools 34
   - Android Emulator
   - Android SDK Platform-Tools

Configura las variables de entorno en `~/.bashrc` o `~/.zshrc`:

```bash
export ANDROID_HOME=$HOME/Android/Sdk
export PATH=$PATH:$ANDROID_HOME/emulator
export PATH=$PATH:$ANDROID_HOME/platform-tools
export PATH=$PATH:$ANDROID_HOME/tools
```

Verifica:

```bash
adb --version
# Android Debug Bridge version 1.x.x
```

### Xcode (solo macOS, para iOS)

Instala Xcode desde la Mac App Store. Después instala las herramientas de línea de comandos:

```bash
xcode-select --install
sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
sudo xcodebuild -runFirstLaunch
```

Instala CocoaPods:

```bash
sudo gem install cocoapods
# o con Homebrew:
brew install cocoapods
```

### React Native CLI

```bash
npm install -g @react-native-community/cli
# Verifica:
npx @react-native-community/cli --version
```

### Resumen de versiones

| Herramienta | Versión mínima | Comando de verificación |
|---|---|---|
| Node.js | 18.0.0 | `node --version` |
| npm | 9.0.0 | `npm --version` |
| Java | 17 | `java -version` |
| Android SDK | Platform 34 | Android Studio SDK Manager |
| CocoaPods | 1.12+ | `pod --version` (macOS) |
| Xcode | 14+ | Xcode → About (macOS) |

---

## 1.2 Crear el proyecto

### Inicializar con TypeScript

```bash
npx @react-native-community/cli init ShopApp --template react-native-template-typescript
cd ShopApp
```

El template de TypeScript genera la estructura inicial con `tsconfig.json`, `App.tsx` y los tipos necesarios.

### Verificar que compila

**Android:**

```bash
# Inicia el emulador primero desde Android Studio:
# AVD Manager → tu dispositivo virtual → Play
npx react-native run-android
```

**iOS (solo macOS):**

```bash
npx react-native run-ios
```

Si el emulador muestra la pantalla de bienvenida de React Native la instalación base es correcta.

### Notas de configuración por plataforma

**Android — `android/local.properties`**

Si Android Studio no encuentra el SDK, crea el archivo manualmente:

```properties
sdk.dir=/home/TU_USUARIO/Android/Sdk
# macOS:
# sdk.dir=/Users/TU_USUARIO/Library/Android/sdk
```

**Android — Permisos de internet**

Verifica que `android/app/src/main/AndroidManifest.xml` incluya:

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

**Android — Cleartext traffic (para desarrollo con HTTP)**

El backend Django corre en HTTP durante el desarrollo. Agrega en `AndroidManifest.xml` dentro de `<application>`:

```xml
<application
  android:usesCleartextTraffic="true"
  ...>
```

**iOS — ATS para desarrollo local**

En `ios/ShopApp/Info.plist` agrega para permitir conexiones HTTP al emulador:

```xml
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <true/>
</dict>
```

> **Nota:** Estas configuraciones son solo para desarrollo. En producción se usa HTTPS y se eliminan estas excepciones.

---

## 1.3 Dependencias

Instala todas las dependencias que se usan a lo largo de la serie. Es mejor instalarlas todas ahora para evitar reinstalaciones entre módulos.

### Navegación

```bash
npm install @react-navigation/native @react-navigation/stack @react-navigation/bottom-tabs
npm install react-native-screens react-native-safe-area-context
npm install react-native-gesture-handler react-native-reanimated
```

### HTTP y almacenamiento seguro

```bash
npm install axios
npm install react-native-encrypted-storage
```

### Estado global

```bash
npm install zustand
```

### UI — NativeWind (TailwindCSS para React Native)

```bash
npm install nativewind
npm install --save-dev tailwindcss
```

### Imágenes

```bash
npm install react-native-fast-image
npm install react-native-image-picker
```

### Utilidades de fecha

```bash
npm install date-fns
```

### Resumen de un solo comando (alternativa)

```bash
npm install \
  axios \
  react-native-encrypted-storage \
  @react-navigation/native \
  @react-navigation/stack \
  @react-navigation/bottom-tabs \
  react-native-screens \
  react-native-safe-area-context \
  react-native-gesture-handler \
  react-native-reanimated \
  zustand \
  nativewind \
  react-native-fast-image \
  react-native-image-picker \
  date-fns

npm install --save-dev tailwindcss
```

### Enlace de módulos nativos — iOS

```bash
cd ios && pod install && cd ..
```

### Verificar dependencias instaladas

```bash
npm list --depth=0
```

Deberías ver todos los paquetes listados sin errores.

---

## 1.4 Configurar NativeWind

NativeWind permite usar clases de TailwindCSS en React Native. En esta serie se usa principalmente el sistema de colores de `colors.ts` con estilos inline, pero NativeWind se configura para los módulos que lo usen directamente.

### Inicializar TailwindCSS

```bash
npx tailwindcss init
```

Esto genera `tailwind.config.js` en la raíz del proyecto.

### Configurar `tailwind.config.js`

Reemplaza el contenido generado:

```javascript
// tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './App.{js,jsx,ts,tsx}',
    './src/**/*.{js,jsx,ts,tsx}',
  ],
  presets: [require('nativewind/preset')],
  theme: {
    extend: {
      colors: {
        background: '#0A0A0F',
        surface:    '#13131A',
        surface2:   '#1C1C26',
        accent:     '#6C63FF',
        success:    '#22C55E',
        error:      '#EF4444',
        border:     '#2A2A38',
        borderLight:'#3A3A50',
      },
    },
  },
  plugins: [],
};
```

### Crear `global.css`

En la raíz del proyecto (junto a `App.tsx`):

```css
/* global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### Actualizar `babel.config.js`

```javascript
// babel.config.js
module.exports = {
  presets: ['module:@react-native/babel-preset'],
  plugins: [
    'nativewind/babel',
    'react-native-reanimated/plugin', // siempre al final
  ],
};
```

> **Importante:** `react-native-reanimated/plugin` debe ser el último plugin en la lista.

### Declaraciones de TypeScript para NativeWind

Crea el archivo `nativewind-env.d.ts` en la raíz:

```typescript
// nativewind-env.d.ts
/// <reference types="nativewind/types" />
```

### Declaración del módulo CSS

Crea `src/types/global.d.ts` (o en la raíz):

```typescript
// global.d.ts
declare module '*.css' {
  const content: Record<string, string>;
  export default content;
}
```

### Importar `global.css` en `App.tsx`

```tsx
// App.tsx — línea al inicio de los imports
import './global.css';
```

### Verificar que Metro reconoce los estilos

Después de actualizar `babel.config.js` limpia la caché de Metro:

```bash
npx react-native start --reset-cache
```

---

## 1.5 Estructura de carpetas

Crea la estructura de directorios antes de añadir código. Esta organización sigue Clean Architecture con capas globales: data, domain y presentation.

### Crear los directorios

```bash
mkdir -p src/data/{api,storage}
mkdir -p src/domain/{model,store}
mkdir -p src/presentation/{screens/{auth,catalog,cart,orders,profile,admin},components,navigation}
mkdir -p src/theme
mkdir -p src/core/config
```

### Árbol completo de la estructura

```
ShopApp/
├── android/
├── ios/
├── src/
│   ├── data/
│   │   ├── api/
│   │   │   ├── axios.client.ts       ← Cliente HTTP con interceptores JWT
│   │   │   ├── api.error.ts          ← Clase ApiError y parseApiError
│   │   │   ├── auth.api.ts           ← login, register, logout
│   │   │   ├── catalog.api.ts        ← categorías y productos
│   │   │   ├── orders.api.ts         ← CRUD de pedidos
│   │   │   └── users.api.ts          ← CRUD de usuarios (admin)
│   │   └── storage/
│   │       └── secure.storage.ts     ← Wrapper de EncryptedStorage
│   ├── domain/
│   │   ├── model/
│   │   │   ├── auth.types.ts
│   │   │   ├── catalog.types.ts
│   │   │   ├── orders.types.ts
│   │   │   └── user.types.ts
│   │   └── store/
│   │       ├── auth.store.ts         ← Zustand store de autenticación
│   │       ├── catalog.store.ts
│   │       ├── orders.store.ts
│   │       ├── profile.store.ts
│   │       └── admin.store.ts
│   ├── presentation/
│   │   ├── screens/
│   │   │   ├── auth/
│   │   │   │   ├── LoginScreen.tsx
│   │   │   │   └── RegisterScreen.tsx
│   │   │   ├── catalog/
│   │   │   │   ├── CatalogScreen.tsx
│   │   │   │   └── ProductDetailScreen.tsx
│   │   │   ├── cart/
│   │   │   │   └── CartScreen.tsx
│   │   │   ├── orders/
│   │   │   │   ├── OrdersScreen.tsx
│   │   │   │   └── OrderDetailScreen.tsx
│   │   │   ├── profile/
│   │   │   │   └── ProfileScreen.tsx
│   │   │   └── admin/
│   │   │       ├── AdminProductsScreen.tsx
│   │   │       ├── AdminCategoriesScreen.tsx
│   │   │       ├── AdminOrdersScreen.tsx
│   │   │       └── AdminUsersScreen.tsx
│   │   ├── components/
│   │   │   ├── ProductCard.tsx
│   │   │   ├── CategoryChip.tsx
│   │   │   └── StatusBadge.tsx
│   │   └── navigation/
│   │       ├── AppNavigator.tsx      ← Navegador principal (tab + stack)
│   │       ├── AuthNavigator.tsx     ← Stack de autenticación
│   │       └── types.ts              ← Tipos de rutas de navegación
│   ├── theme/
│   │   └── colors.ts                 ← Design System de colores
│   └── core/
│       └── config/
│           └── api.config.ts         ← URL base del backend
├── App.tsx
├── global.css
├── babel.config.js
├── tailwind.config.js
├── tsconfig.json
└── nativewind-env.d.ts
```

### Principios de la estructura

- **`src/data/`**: Acceso a datos — API calls y almacenamiento local. No importa de `domain/` ni `presentation/`.
- **`src/domain/`**: Lógica de negocio — tipos/modelos y stores Zustand. Importa solo de `data/`.
- **`src/presentation/`**: UI — pantallas, componentes y navegación. Importa de `domain/` y `data/`.
- **`src/theme/`**: Design system compartido por toda la app.
- **`src/core/`**: Configuración global (URL, constantes).
- **Regla de importación**: `presentation → domain → data`. Nunca hacia arriba.

---

## 1.6 Design System

### Colors — `src/theme/colors.ts`

Este archivo define la paleta completa usada en toda la serie. Todos los módulos importan de aquí.

```typescript
// src/theme/colors.ts

export const Colors = {
  // Fondos
  background: '#0A0A0F',   // Fondo principal de la app (negro profundo)
  surface:    '#13131A',   // Cards, modales, panels
  surface2:   '#1C1C26',   // Elementos dentro de cards (hover, inputs)

  // Acento principal
  accent:     '#6C63FF',   // Violeta — botones primarios, links, highlights

  // Textos
  textPrimary:   '#F0F0FF', // Títulos, texto principal
  textSecondary: '#8888A0', // Subtítulos, labels, texto secundario

  // Bordes
  border:      '#2A2A38',  // Bordes de cards y containers
  borderLight: '#3A3A50',  // Separadores, dividers sutiles

  // Estados
  success: '#22C55E',  // Éxito, stock disponible, pedido confirmado
  error:   '#EF4444',  // Error, stock agotado, fallo de conexión
  warning: '#F59E0B',  // Advertencias, stock bajo
  info:    '#3B82F6',  // Información, badges neutrales
} as const;

export type ColorKey = keyof typeof Colors;
```

### Uso del Design System

```typescript
// Ejemplo de uso en cualquier pantalla
import {Colors} from '../../theme/colors';

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  card: {
    backgroundColor: Colors.surface,
    borderWidth: 1,
    borderColor: Colors.border,
    borderRadius: 16,
    padding: 16,
  },
  title: {
    color: Colors.textPrimary,
    fontSize: 18,
    fontWeight: '700',
  },
  subtitle: {
    color: Colors.textSecondary,
    fontSize: 14,
  },
  accentButton: {
    backgroundColor: Colors.accent,
    borderRadius: 12,
    padding: 14,
    alignItems: 'center',
  },
});
```

### API Config — `src/core/config/api.config.ts`

La IP `10.0.2.2` es el alias que el emulador de Android usa para referirse a `localhost` de la máquina host. En iOS el simulador puede usar `localhost` directamente.

```typescript
// src/core/config/api.config.ts
import {Platform} from 'react-native';

/**
 * URL base del backend Django REST Framework.
 *
 * Android Emulator: 10.0.2.2 → localhost de la máquina host
 * iOS Simulator:    localhost → localhost de la máquina host
 * Dispositivo físico: usar la IP local de tu máquina (ej. 192.168.1.x)
 */
export const API_BASE_URL: string =
  Platform.OS === 'android'
    ? 'http://10.0.2.2:8000/api'
    : 'http://localhost:8000/api';
```

> **Dispositivo físico:** Si ejecutas la app en un dispositivo Android real (no emulador), cambia la IP por la IP local de tu computadora en la red, por ejemplo `http://192.168.1.100:8000/api`. Ambos dispositivos deben estar en la misma red WiFi.

---

## 1.7 Configurar React Navigation

### Tipos de rutas — `src/presentation/navigation/types.ts`

Define los parámetros de cada ruta para tener navegación con tipos:

```typescript
// src/presentation/navigation/types.ts
import {StackScreenProps} from '@react-navigation/stack';
import {BottomTabScreenProps} from '@react-navigation/bottom-tabs';

// Stack de autenticación
export type AuthStackParamList = {
  Login:    undefined;
  Register: undefined;
};

// Tabs principales (usuario autenticado)
export type MainTabParamList = {
  Catalog: undefined;
  Orders:  undefined;
  Profile: undefined;
  Admin:   undefined; // solo visible para is_staff
};

// Stack del catálogo
export type CatalogStackParamList = {
  CatalogHome:   undefined;
  ProductDetail: {productId: number};
};

// Stack de pedidos
export type OrdersStackParamList = {
  OrdersList:  undefined;
  OrderDetail: {orderId: number};
};

// Stack de admin
export type AdminStackParamList = {
  AdminHome:    undefined;
  AdminUsers:   undefined;
  AdminCatalog: undefined;
  AdminOrders:  undefined;
};

// Props helpers
export type AuthScreenProps<T extends keyof AuthStackParamList> =
  StackScreenProps<AuthStackParamList, T>;

export type MainTabProps<T extends keyof MainTabParamList> =
  BottomTabScreenProps<MainTabParamList, T>;
```

### Auth Navigator — `src/presentation/navigation/AuthNavigator.tsx`

```tsx
// src/presentation/navigation/AuthNavigator.tsx
import React from 'react';
import {createStackNavigator} from '@react-navigation/stack';
import {Colors} from '../../theme/colors';
import {AuthStackParamList} from './types';

// Pantallas placeholder para el Módulo 1
function LoginPlaceholder() {
  const {View, Text} = require('react-native');
  return (
    <View style={{flex: 1, backgroundColor: Colors.background, justifyContent: 'center', alignItems: 'center'}}>
      <Text style={{color: Colors.accent, fontSize: 24, fontWeight: '700'}}>Login</Text>
      <Text style={{color: Colors.textSecondary, marginTop: 8}}>Módulo 3</Text>
    </View>
  );
}

function RegisterPlaceholder() {
  const {View, Text} = require('react-native');
  return (
    <View style={{flex: 1, backgroundColor: Colors.background, justifyContent: 'center', alignItems: 'center'}}>
      <Text style={{color: Colors.accent, fontSize: 24, fontWeight: '700'}}>Register</Text>
      <Text style={{color: Colors.textSecondary, marginTop: 8}}>Módulo 3</Text>
    </View>
  );
}

const Stack = createStackNavigator<AuthStackParamList>();

export function AuthNavigator() {
  return (
    <Stack.Navigator
      screenOptions={{
        headerShown: false,
        cardStyle: {backgroundColor: Colors.background},
      }}>
      <Stack.Screen name="Login"    component={LoginPlaceholder} />
      <Stack.Screen name="Register" component={RegisterPlaceholder} />
    </Stack.Navigator>
  );
}
```

### App Navigator — `src/presentation/navigation/AppNavigator.tsx`

```tsx
// src/presentation/navigation/AppNavigator.tsx
import React from 'react';
import {View, Text, ActivityIndicator} from 'react-native';
import {createBottomTabNavigator} from '@react-navigation/bottom-tabs';
import {Colors} from '../../theme/colors';
import {MainTabParamList} from './types';
import {AuthNavigator} from './AuthNavigator';

// Pantallas placeholder
function CatalogPlaceholder() {
  return (
    <View style={{flex: 1, backgroundColor: Colors.background, justifyContent: 'center', alignItems: 'center'}}>
      <Text style={{color: Colors.accent, fontSize: 24, fontWeight: '700'}}>Catálogo</Text>
      <Text style={{color: Colors.textSecondary, marginTop: 8}}>Módulos 4-5</Text>
    </View>
  );
}

function OrdersPlaceholder() {
  return (
    <View style={{flex: 1, backgroundColor: Colors.background, justifyContent: 'center', alignItems: 'center'}}>
      <Text style={{color: Colors.accent, fontSize: 24, fontWeight: '700'}}>Pedidos</Text>
      <Text style={{color: Colors.textSecondary, marginTop: 8}}>Módulos 6-7</Text>
    </View>
  );
}

function ProfilePlaceholder() {
  return (
    <View style={{flex: 1, backgroundColor: Colors.background, justifyContent: 'center', alignItems: 'center'}}>
      <Text style={{color: Colors.accent, fontSize: 24, fontWeight: '700'}}>Perfil</Text>
      <Text style={{color: Colors.textSecondary, marginTop: 8}}>Módulo 8</Text>
    </View>
  );
}

const Tab = createBottomTabNavigator<MainTabParamList>();

// Navigator para usuarios autenticados
function MainNavigator() {
  return (
    <Tab.Navigator
      screenOptions={{
        headerShown: false,
        tabBarStyle: {
          backgroundColor: Colors.surface,
          borderTopColor:  Colors.border,
          borderTopWidth:  1,
          paddingBottom:   8,
          paddingTop:      8,
          height:          64,
        },
        tabBarActiveTintColor:   Colors.accent,
        tabBarInactiveTintColor: Colors.textSecondary,
        tabBarLabelStyle: {
          fontSize:   11,
          fontWeight: '600',
          marginTop:  2,
        },
      }}>
      <Tab.Screen
        name="Catalog"
        component={CatalogPlaceholder}
        options={{tabBarLabel: 'Catálogo'}}
      />
      <Tab.Screen
        name="Orders"
        component={OrdersPlaceholder}
        options={{tabBarLabel: 'Pedidos'}}
      />
      <Tab.Screen
        name="Profile"
        component={ProfilePlaceholder}
        options={{tabBarLabel: 'Perfil'}}
      />
    </Tab.Navigator>
  );
}

/**
 * AppNavigator — decide qué stack mostrar según el estado de sesión.
 *
 * En el Módulo 1 siempre muestra el AuthNavigator.
 * En el Módulo 3 se conectará al auth.store de Zustand para
 * decidir dinámicamente entre AuthNavigator y MainNavigator.
 */
export function AppNavigator() {
  // TODO Módulo 3: const {user, loading} = useAuthStore();
  const isAuthenticated = false; // placeholder

  return isAuthenticated ? <MainNavigator /> : <AuthNavigator />;
}
```

---

## 1.8 App.tsx base

Reemplaza el contenido de `App.tsx` con la configuración base de la serie:

```tsx
// App.tsx
import React from 'react';
import {StatusBar} from 'react-native';
import {NavigationContainer} from '@react-navigation/native';
import {SafeAreaProvider} from 'react-native-safe-area-context';
import {GestureHandlerRootView} from 'react-native-gesture-handler';
import {AppNavigator} from './src/presentation/navigation/AppNavigator';
import {Colors} from './src/theme/colors';
import './global.css';

export default function App() {
  return (
    <GestureHandlerRootView style={{flex: 1}}>
      <SafeAreaProvider>
        <NavigationContainer>
          <StatusBar
            barStyle="light-content"
            backgroundColor={Colors.background}
          />
          <AppNavigator />
        </NavigationContainer>
      </SafeAreaProvider>
    </GestureHandlerRootView>
  );
}
```

### Explicación de los wrappers

| Wrapper | Por qué es necesario |
|---|---|
| `GestureHandlerRootView` | Requerido por `react-native-gesture-handler` para que los gestos funcionen en Android |
| `SafeAreaProvider` | Provee información del safe area (notch, barra de navegación) a toda la app |
| `NavigationContainer` | Contexto de React Navigation. Gestiona el estado de la navegación y el historial |
| `StatusBar` | Configura el estilo de la barra de estado del sistema para que sea coherente con el tema oscuro |

### Pantalla de bienvenida para verificación

Mientras construyes la navegación en el Módulo 1 puedes usar esta pantalla de bienvenida directamente en `App.tsx` para verificar que los colores y la configuración base funcionan:

```tsx
// App.tsx — versión de verificación del Módulo 1
import React from 'react';
import {
  StatusBar,
  View,
  Text,
  ScrollView,
  StyleSheet,
} from 'react-native';
import {SafeAreaProvider, SafeAreaView} from 'react-native-safe-area-context';
import {GestureHandlerRootView} from 'react-native-gesture-handler';
import {Colors} from './src/theme/colors';
import {API_BASE_URL} from './src/core/config/api.config';
import './global.css';

export default function App() {
  const checklist = [
    {item: 'TypeScript',                   file: 'tsconfig.json'},
    {item: 'NativeWind',                   file: 'babel.config.js'},
    {item: 'React Navigation',             file: 'src/presentation/navigation/'},
    {item: 'Design System (Colors)',       file: 'src/theme/colors.ts'},
    {item: 'API Config',                   file: 'src/core/config/api.config.ts'},
    {item: 'Estructura Clean Architecture', file: 'src/data/ + src/domain/ + src/presentation/'},
    {item: 'GestureHandler + SafeArea',    file: 'App.tsx'},
    {item: 'axios (instalado)',            file: 'node_modules/axios'},
    {item: 'EncryptedStorage (instalado)', file: 'node_modules/react-native-encrypted-storage'},
    {item: 'Zustand (instalado)',          file: 'node_modules/zustand'},
  ];

  return (
    <GestureHandlerRootView style={{flex: 1}}>
      <SafeAreaProvider>
        <StatusBar barStyle="light-content" backgroundColor={Colors.background} />
        <SafeAreaView style={styles.container}>
          <ScrollView contentContainerStyle={styles.scroll}>

            {/* Header */}
            <View style={styles.header}>
              <Text style={styles.logo}>ShopApp</Text>
              <Text style={styles.logoSub}>React Native</Text>
              <View style={styles.badge}>
                <Text style={styles.badgeText}>Módulo 1 — Setup completado</Text>
              </View>
            </View>

            {/* API URL */}
            <View style={styles.card}>
              <Text style={styles.sectionLabel}>BACKEND URL</Text>
              <View style={styles.row}>
                <Text style={styles.rowLabel}>API_BASE_URL</Text>
                <Text style={styles.rowCode}>{API_BASE_URL}</Text>
              </View>
            </View>

            {/* Checklist */}
            <Text style={styles.sectionLabel}>CHECKLIST MÓDULO 1</Text>
            <View style={styles.card}>
              {checklist.map((entry, i) => (
                <View key={entry.item}>
                  <View style={styles.checkRow}>
                    <Text style={styles.checkMark}>✓</Text>
                    <View style={{flex: 1}}>
                      <Text style={styles.checkItem}>{entry.item}</Text>
                      <Text style={styles.checkFile}>{entry.file}</Text>
                    </View>
                  </View>
                  {i < checklist.length - 1 && (
                    <View style={styles.divider} />
                  )}
                </View>
              ))}
            </View>

            {/* Colores del Design System */}
            <Text style={styles.sectionLabel}>DESIGN SYSTEM</Text>
            <View style={styles.card}>
              {Object.entries(Colors).map(([key, value], i, arr) => (
                <View key={key}>
                  <View style={styles.colorRow}>
                    <View style={[styles.colorSwatch, {backgroundColor: value}]} />
                    <Text style={styles.colorKey}>{key}</Text>
                    <Text style={styles.colorValue}>{value}</Text>
                  </View>
                  {i < arr.length - 1 && (
                    <View style={styles.divider} />
                  )}
                </View>
              ))}
            </View>

            <Text style={styles.footer}>
              Siguiente → Módulo 2: Axios + JWT + EncryptedStorage
            </Text>

          </ScrollView>
        </SafeAreaView>
      </SafeAreaProvider>
    </GestureHandlerRootView>
  );
}

const styles = StyleSheet.create({
  container: {
    flex: 1,
    backgroundColor: Colors.background,
  },
  scroll: {
    padding: 20,
    paddingBottom: 48,
  },
  header: {
    alignItems: 'center',
    marginTop: 24,
    marginBottom: 32,
  },
  logo: {
    fontSize: 44,
    fontWeight: '800',
    color: Colors.accent,
    letterSpacing: -1,
  },
  logoSub: {
    fontSize: 14,
    color: Colors.textSecondary,
    marginTop: 4,
    letterSpacing: 2,
  },
  badge: {
    marginTop: 12,
    backgroundColor: Colors.surface2,
    borderRadius: 20,
    paddingHorizontal: 16,
    paddingVertical: 6,
    borderWidth: 1,
    borderColor: Colors.border,
  },
  badgeText: {
    color: Colors.success,
    fontSize: 12,
    fontWeight: '600',
  },
  card: {
    backgroundColor: Colors.surface,
    borderRadius: 16,
    borderWidth: 1,
    borderColor: Colors.border,
    padding: 16,
    marginBottom: 20,
  },
  sectionLabel: {
    color: Colors.textSecondary,
    fontSize: 11,
    fontWeight: '700',
    letterSpacing: 1.5,
    marginBottom: 10,
    marginTop: 4,
  },
  row: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    alignItems: 'center',
    paddingVertical: 8,
  },
  rowLabel: {
    color: Colors.textSecondary,
    fontSize: 13,
  },
  rowCode: {
    color: Colors.accent,
    fontSize: 12,
    fontFamily: 'monospace',
  },
  checkRow: {
    flexDirection: 'row',
    alignItems: 'flex-start',
    paddingVertical: 10,
    gap: 12,
  },
  checkMark: {
    color: Colors.success,
    fontWeight: '700',
    fontSize: 16,
    marginTop: 1,
  },
  checkItem: {
    color: Colors.textPrimary,
    fontSize: 13,
    fontWeight: '600',
  },
  checkFile: {
    color: Colors.textSecondary,
    fontSize: 11,
    fontFamily: 'monospace',
    marginTop: 2,
  },
  divider: {
    height: 0.5,
    backgroundColor: Colors.borderLight,
  },
  colorRow: {
    flexDirection: 'row',
    alignItems: 'center',
    paddingVertical: 8,
    gap: 12,
  },
  colorSwatch: {
    width: 28,
    height: 28,
    borderRadius: 6,
    borderWidth: 1,
    borderColor: Colors.borderLight,
  },
  colorKey: {
    color: Colors.textPrimary,
    fontSize: 13,
    fontWeight: '600',
    flex: 1,
  },
  colorValue: {
    color: Colors.textSecondary,
    fontSize: 12,
    fontFamily: 'monospace',
  },
  footer: {
    color: Colors.textSecondary,
    fontSize: 13,
    textAlign: 'center',
    marginTop: 8,
    paddingBottom: 16,
  },
});
```

---

## 1.9 Verificar en emulador

### Iniciar el backend de Django

El emulador necesita que el backend esté corriendo para los módulos siguientes. Levántalo ya:

```bash
# En el repositorio de Django
uv run python manage.py runserver 0.0.0.0:8000
```

El flag `0.0.0.0` permite conexiones desde el emulador Android.

### Iniciar Metro Bundler

En una terminal nueva dentro de la carpeta del proyecto:

```bash
cd ShopApp
npx react-native start
```

Mantén Metro corriendo en esta terminal durante todo el desarrollo.

### Ejecutar en Android

```bash
# En otra terminal
npx react-native run-android
```

Si tienes varios emuladores o dispositivos conectados:

```bash
adb devices
# Lista los dispositivos disponibles
npx react-native run-android --deviceId emulator-5554
```

### Ejecutar en iOS (macOS)

```bash
npx react-native run-ios
# Para un simulador específico:
npx react-native run-ios --simulator "iPhone 15 Pro"
```

### Limpiar caché si hay errores

```bash
# Limpiar caché de Metro
npx react-native start --reset-cache

# Limpiar build de Android
cd android && ./gradlew clean && cd ..

# Limpiar build de iOS
cd ios && xcodebuild clean && cd ..
rm -rf ios/Pods ios/Podfile.lock
cd ios && pod install && cd ..
```

### Recarga en caliente

Con la app corriendo en el emulador:

- **Android:** doble tap en `R` o `Ctrl+M` para el menú de desarrollo
- **iOS:** `Cmd+R` para recargar, `Cmd+D` para el menú de desarrollo

### Errores frecuentes en el Módulo 1

| Error | Causa probable | Solución |
|---|---|---|
| `JAVA_HOME not found` | Java no configurado | Configura `JAVA_HOME` en el perfil de la shell |
| `SDK location not found` | `local.properties` faltante | Crea `android/local.properties` con `sdk.dir` |
| `Unable to connect to Metro` | Metro no está corriendo | Inicia `npx react-native start` |
| `Invariant Violation: GestureHandlerRootView` | Falta el wrapper | Verifica `App.tsx` |
| `Unable to resolve module nativewind` | NativeWind no instalado | `npm install nativewind` |
| `babel-plugin-module-resolver` | Plugin de Babel faltante | Verifica `babel.config.js` |
| `pod install failed` | CocoaPods desactualizado | `sudo gem update cocoapods` |

---

## ✅ Checkpoint Módulo 1

Verifica cada punto antes de continuar al Módulo 2:

### Archivos creados

| Archivo | Descripción | Estado esperado |
|---|---|---|
| `tailwind.config.js` | Configuración TailwindCSS con colores del Design System | Existe en la raíz |
| `babel.config.js` | Incluye `nativewind/babel` y `react-native-reanimated/plugin` | Actualizado |
| `global.css` | `@tailwind base/components/utilities` | Existe en la raíz |
| `nativewind-env.d.ts` | Referencia a tipos de NativeWind | Existe en la raíz |
| `src/theme/colors.ts` | Objeto `Colors` con 11 colores | Creado |
| `src/core/config/api.config.ts` | `API_BASE_URL` con detección de plataforma | Creado |
| `src/presentation/navigation/types.ts` | Tipos de rutas con TypeScript | Creado |
| `src/presentation/navigation/AuthNavigator.tsx` | Stack de Login/Register | Creado |
| `src/presentation/navigation/AppNavigator.tsx` | Tabs + lógica de sesión (placeholder) | Creado |
| `App.tsx` | Wrappers GestureHandler + SafeArea + Navigation | Actualizado |

### Estructura de directorios

| Directorio | Estado esperado |
|---|---|
| `src/data/api/` | Directorio creado (vacío por ahora) |
| `src/data/storage/` | Directorio creado (vacío por ahora) |
| `src/domain/model/` | Directorio creado (vacío por ahora) |
| `src/domain/store/` | Directorio creado (vacío por ahora) |
| `src/presentation/screens/auth/` | Directorio creado (vacío por ahora) |
| `src/presentation/screens/catalog/` | Directorio creado (vacío por ahora) |
| `src/presentation/screens/orders/` | Directorio creado (vacío por ahora) |
| `src/presentation/screens/profile/` | Directorio creado (vacío por ahora) |
| `src/presentation/screens/admin/` | Directorio creado (vacío por ahora) |
| `src/presentation/navigation/` | Archivos AppNavigator, AuthNavigator, types |

### Verificaciones visuales

| Verificación | Resultado esperado |
|---|---|
| La app compila sin errores | Metro no muestra errores en rojo |
| Pantalla de bienvenida visible | Logo "ShopApp" en violeta sobre fondo oscuro |
| Lista de checklist visible | 10 ítems con ✓ en verde |
| Paleta de colores visible | 11 swatches de color del Design System |
| `API_BASE_URL` visible | `http://10.0.2.2:8000/api` en Android |
| Sin warnings de TypeScript | `npx tsc --noEmit` no reporta errores |

### Verificar TypeScript

```bash
npx tsc --noEmit
# Sin output = sin errores de tipos
```

### Dependencias instaladas

```bash
npm list axios react-native-encrypted-storage @react-navigation/native zustand nativewind --depth=0
```

Todas deben aparecer en la lista sin el sufijo `(missing)`.

---

## Resumen

| Elemento | Estado |
|---|---|
| React Native CLI con TypeScript template | ✅ |
| Android Studio + SDK configurados | ✅ |
| Todas las dependencias de la serie instaladas | ✅ |
| NativeWind configurado (tailwind.config.js + babel + global.css) | ✅ |
| Estructura Clean Architecture creada | ✅ |
| Design System — `src/theme/colors.ts` | ✅ |
| API Config — `src/core/config/api.config.ts` | ✅ |
| React Navigation configurado (Stack + Tabs) | ✅ |
| `App.tsx` con GestureHandler + SafeArea + NavigationContainer | ✅ |
| App corriendo en emulador sin errores | ✅ |

**Siguiente módulo →** M2: Axios + Interceptores JWT + EncryptedStorage + Repositorios base
