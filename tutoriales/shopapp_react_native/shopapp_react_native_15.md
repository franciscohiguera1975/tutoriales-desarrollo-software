# ShopApp React Native — Módulo 15

## Distribución — Build de producción para Android e iOS

**Duración estimada:** 2–3 horas
**Prerequisitos:** Módulos 1–14 completados. La app debe pasar todos los tests del Módulo 13.

---

> **Objetivo**
> Preparar la app para distribución: configurar variables de entorno por ambiente, generar el build firmado de Android (APK/AAB) y el archivo IPA de iOS, probar el build de producción en dispositivo físico, y conocer el proceso de publicación en Google Play Store y Apple App Store.

---

> **Checkpoint final**
> Al terminar este módulo deberás poder:
> - Cambiar entre la URL de la API de desarrollo y producción sin modificar código
> - Generar un APK de release instalable en Android
> - Generar un AAB listo para subir a Google Play Console
> - Generar un IPA de release desde Xcode
> - Conocer los pasos para publicar en Play Store y App Store

---

## 15.1 Variables de entorno

**Archivo:** `src/core/config/api.config.ts`

Hay dos enfoques populares para manejar variables de entorno en React Native: `react-native-config` (archivo `.env`) y la configuración manual por esquema de build. Se describe el enfoque con `react-native-config` por ser el más portable.

### Instalar react-native-config

```bash
npm install react-native-config
cd ios && pod install && cd ..
```

### Archivos .env

Crea en la raíz del proyecto:

```bash
# .env               → desarrollo local
API_BASE_URL=http://10.0.2.2:8000
APP_ENV=development
```

```bash
# .env.production    → producción
API_BASE_URL=https://api.myshopapp.com
APP_ENV=production
```

> **10.0.2.2** es la IP especial del emulador Android para acceder al `localhost` del host. En un dispositivo físico en la misma red, usa la IP local de tu máquina (p. ej. `192.168.1.X`).

### Configurar Android para leer .env

Edita `android/app/build.gradle`:

```groovy
// Al inicio del archivo, después de 'apply plugin':
project.ext.envConfigFiles = [
    debug: ".env",
    release: ".env.production",
]
apply from: project(':react-native-config').projectDir.getPath() + "/dotenv.gradle"
```

### Configurar iOS para leer .env

En Xcode: selecciona el target → Build Settings → busca "User-Defined Settings" → agrega:
- Scheme `Debug`: `$(PROJECT_DIR)/.env`
- Scheme `Release`: `$(PROJECT_DIR)/.env.production`

### Archivo de configuración de la API

```typescript
// src/core/config/api.config.ts
import Config from 'react-native-config';

// Validar que la variable de entorno esté definida
if (!Config.API_BASE_URL) {
  throw new Error(
    'API_BASE_URL no está definida. Verifica tu archivo .env',
  );
}

export const API_BASE_URL: string = Config.API_BASE_URL;
export const APP_ENV: string = Config.APP_ENV || 'development';

// Tiempo máximo de espera para peticiones (en ms)
export const API_TIMEOUT = 15_000;

// Indica si estamos en modo desarrollo
export const IS_DEV = APP_ENV === 'development';

// Usado en los módulos de API:
// import { API_BASE_URL } from '../../core/config/api.config';
// En api.client.ts: baseURL: API_BASE_URL
```

---

## 15.2 Build de Android

### Paso 1: Generar el Keystore

El keystore es el certificado que firma la app. **Guárdalo en un lugar seguro; sin él no podrás publicar actualizaciones de la app.**

```bash
keytool -genkeypair \
  -v \
  -storetype PKCS12 \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -storepass MI_PASSWORD_SEGURA \
  -keypass MI_PASSWORD_SEGURA \
  -alias shopapp-key-alias \
  -keystore android/app/shopapp-release.keystore \
  -dname "CN=ShopApp, OU=MobileTeam, O=MiEmpresa, L=Quito, S=Pichincha, C=EC"
```

Reemplaza `MI_PASSWORD_SEGURA` con una contraseña fuerte y guárdala.

**Agrega el keystore al .gitignore:**

```
# .gitignore
android/app/*.keystore
android/app/*.jks
.env.production
```

### Paso 2: Configurar las credenciales de firma

Crea `android/gradle.properties` (o agrega al existente):

```properties
# Credenciales del keystore — NO commitear este archivo
MYAPP_UPLOAD_STORE_FILE=shopapp-release.keystore
MYAPP_UPLOAD_KEY_ALIAS=shopapp-key-alias
MYAPP_UPLOAD_STORE_PASSWORD=MI_PASSWORD_SEGURA
MYAPP_UPLOAD_KEY_PASSWORD=MI_PASSWORD_SEGURA
```

### Paso 3: Configurar signing en build.gradle

Edita `android/app/build.gradle`:

```groovy
android {
    // ...

    signingConfigs {
        release {
            if (project.hasProperty('MYAPP_UPLOAD_STORE_FILE')) {
                storeFile file(MYAPP_UPLOAD_STORE_FILE)
                storePassword MYAPP_UPLOAD_STORE_PASSWORD
                keyAlias MYAPP_UPLOAD_KEY_ALIAS
                keyPassword MYAPP_UPLOAD_KEY_PASSWORD
            }
        }
    }

    buildTypes {
        release {
            signingConfig signingConfigs.release
            minifyEnabled enableProguardInReleaseBuilds
            proguardFiles getDefaultProguardFile("proguard-android.txt"), "proguard-rules.pro"
            // Usar las variables de entorno de producción
        }
        debug {
            signingConfig signingConfigs.debug
        }
    }
}
```

### Paso 4: Generar el APK de release

```bash
# APK para instalación directa (testing en dispositivos)
npx react-native build-android --mode=release

# Alternativa con Gradle directamente:
cd android && ./gradlew assembleRelease

# El APK estará en:
# android/app/build/outputs/apk/release/app-release.apk
```

### Paso 5: Generar el AAB (para Play Store)

Google Play requiere AAB (Android App Bundle) en lugar de APK:

```bash
cd android && ./gradlew bundleRelease

# El AAB estará en:
# android/app/build/outputs/bundle/release/app-release.aab
```

---

## 15.3 Configurar build.gradle para producción

Optimizaciones adicionales para el build de release:

```groovy
// android/app/build.gradle
android {
    buildTypes {
        release {
            // Habilitar minificación de código JavaScript (reduce tamaño del bundle)
            minifyEnabled true

            // Reglas de ProGuard para mantener clases necesarias en runtime
            proguardFiles getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"

            // Comprimir los recursos no utilizados
            shrinkResources true
        }
    }

    // Versión del bundle de JS generado por Metro
    bundleConfig {
        // ...
    }
}
```

### Reglas de ProGuard

Edita `android/app/proguard-rules.pro`:

```proguard
# Mantener clases de React Native
-keep class com.facebook.react.** { *; }
-keep class com.facebook.jni.** { *; }

# Mantener clases de OkHttp (red)
-dontwarn okhttp3.**
-dontwarn okio.**
-keep class okhttp3.** { *; }
-keep interface okhttp3.** { *; }

# Mantener clases de Hermes (motor de JavaScript)
-keep class com.facebook.hermes.** { *; }
-keep class com.facebook.react.turbomodule.** { *; }

# EncryptedStorage
-keep class androidx.security.crypto.** { *; }

# FastImage
-keep class com.dylanvann.fastimage.** { *; }
```

---

## 15.4 Build de iOS

### Prerequisitos

- Mac con Xcode 15 o superior instalado
- Cuenta de Apple Developer (USD 99/año) para distribución real
- Dispositivo registrado en Apple Developer Portal o TestFlight para pruebas

### Paso 1: Configurar el Bundle Identifier

En Xcode → selecciona el proyecto → target → General:
- Bundle Identifier: `com.tuempresa.shopapp` (debe ser único en App Store)

### Paso 2: Configurar Code Signing

En Xcode → target → Signing & Capabilities:
- Team: selecciona tu Apple Developer Team
- Signing Certificate: "Apple Distribution" (para release)
- Provisioning Profile: "App Store" (generado automáticamente o manual)

### Paso 3: Configurar el Scheme de Release

1. En Xcode, selecciona el scheme (junto al selector de dispositivo)
2. Click en "Edit Scheme..."
3. Asegura que la acción "Archive" use la configuración "Release"

### Paso 4: Generar el Archive

1. En Xcode, selecciona "Any iOS Device (arm64)" como destino
2. Menú Product → Archive
3. Esperar a que compile (puede tardar 5–15 minutos)
4. Se abre Xcode Organizer con el archive generado

### Paso 5: Exportar el IPA

En Xcode Organizer (Window → Organizer → Archives):

- Para **TestFlight / App Store**: "Distribute App" → "App Store Connect" → "Upload"
- Para **Ad Hoc** (instalación directa en dispositivos registrados): "Distribute App" → "Ad Hoc"
- Para **Enterprise**: "Distribute App" → "Enterprise"

El IPA se exporta a la carpeta que selecciones.

---

## 15.5 Pruebas del build de release en dispositivo

### Android — instalar APK directamente

```bash
# Con el dispositivo físico conectado por USB (USB Debugging habilitado):
adb install android/app/build/outputs/apk/release/app-release.apk

# Verificar que la URL de la API sea la de producción
# (Abrir la app, ir a cualquier pantalla que haga fetch y revisar los logs)
adb logcat | grep -i "shopapp\|okhttp\|axios"
```

### Verificar la URL de producción

Agrega temporalmente un log en `api.config.ts` solo en desarrollo:

```typescript
if (IS_DEV) {
  console.log('[Config] API_BASE_URL:', API_BASE_URL);
}
```

Esto imprimirá la URL en desarrollo pero no en el build de producción (ya que `IS_DEV` será `false`).

### Checklist de pruebas antes de publicar

- [ ] Login y registro funcionan correctamente
- [ ] Las imágenes de productos cargan desde la URL de producción
- [ ] El flujo completo de compra (carrito → checkout → pedido) funciona
- [ ] Las notificaciones push funcionan (si aplica)
- [ ] El build no muestra logs de desarrollo ni pantallas de debug
- [ ] La app maneja correctamente la ausencia de conexión a internet
- [ ] El tamaño del APK/AAB es razonable (< 50 MB para la mayoría de apps)

```bash
# Ver tamaño del APK generado:
ls -lh android/app/build/outputs/apk/release/app-release.apk

# Ver tamaño del AAB:
ls -lh android/app/build/outputs/bundle/release/app-release.aab
```

---

## 15.6 Publicar en Play Store

### Requisitos previos

- Cuenta de Google Play Console (USD 25 de registro, pago único)
- AAB generado y firmado (`app-release.aab`)
- Capturas de pantalla de la app (mínimo 2, recomendado 4–8)
- Ícono de la app: 512×512 px PNG (sin transparencia)
- Descripción corta (< 80 caracteres) y descripción completa (< 4000 caracteres)
- Clasificación de contenido (formulario de Google Play)

### Pasos en Google Play Console

1. Accede a [play.google.com/console](https://play.google.com/console)
2. "Crear aplicación" → elige el país y el idioma predeterminado
3. En el menú lateral: "Producción" → "Versiones" → "Crear versión"
4. Sube el archivo `.aab`
5. Agrega notas de la versión (Changelog)
6. Completa todas las secciones del formulario:
   - Configuración de la app (categoría, etiquetas de contenido)
   - Ficha de Play Store (capturas, descripción, ícono)
   - Clasificación de contenido
   - Precio y distribución (gratuita o de pago)
7. "Revisar versión" → corrija cualquier problema señalado
8. "Publicar versión" — la revisión de Google tarda 1–7 días

### Actualizaciones

Para actualizaciones, incrementa el `versionCode` en `android/app/build.gradle`:

```groovy
android {
    defaultConfig {
        versionCode 2        // Entero incremental, nunca puede bajar
        versionName "1.1.0"  // Versión legible para el usuario
    }
}
```

---

## 15.7 Publicar en App Store

### Requisitos previos

- Cuenta de Apple Developer (USD 99/año)
- IPA exportado desde Xcode Organizer
- Capturas de pantalla para todos los tamaños de pantalla requeridos (iPhone 6.5", 5.5" y iPad si aplica)
- Ícono de 1024×1024 px PNG (sin transparencia, sin esquinas redondeadas)
- Descripción de la app, palabras clave (< 100 caracteres), categorías

### Subir el IPA

**Opción A — Desde Xcode Organizer (recomendado):**
1. Window → Organizer → selecciona el archive
2. "Distribute App" → "App Store Connect" → "Upload"
3. Xcode sube automáticamente el IPA a App Store Connect

**Opción B — Con Transporter (app de Mac):**
1. Descarga Transporter desde la Mac App Store (gratis)
2. Inicia sesión con tu Apple ID de desarrollador
3. Arrastra el IPA exportado a Transporter
4. Click en "Deliver" para subir

### Pasos en App Store Connect

1. Accede a [appstoreconnect.apple.com](https://appstoreconnect.apple.com)
2. "Mis apps" → "+" → "Nueva app"
3. Completa la información: nombre, idioma, Bundle ID, SKU
4. En la sección "Build", selecciona el build recién subido
5. Completa la ficha de la App Store:
   - Descripción, capturas de pantalla, vista previa de la app
   - Palabras clave, URL de soporte, URL de política de privacidad (obligatoria)
   - Clasificación de edad
   - Información de revisión (cuenta de prueba si la app requiere login)
6. "Agregar para revisión" → Apple tarda entre 24 horas y 3 días en la primera revisión

### Actualizaciones de iOS

Para nuevas versiones, incrementa el `CFBundleVersion` y `CFBundleShortVersionString` en `ios/<App>/Info.plist` o en Xcode → target → General → Identity.

---

> **Resumen del proyecto completo — Módulos 1–15**
>
> | Módulo | Tema | Resultado |
> |--------|------|-----------|
> | 01 | Configuración del proyecto | Proyecto RN con TypeScript, estructura de carpetas, dependencias |
> | 02 | Autenticación | Login/Registro, JWT, EncryptedStorage, AuthStore |
> | 03 | Navegación | React Navigation, stacks de Auth y Main, tab navigator |
> | 04 | Catálogo | Lista de productos y categorías, CatalogStore, filtros |
> | 05 | Detalle de producto | ProductDetailScreen, ProductDetailStore, SKU, stock |
> | 06 | Carrito de compras | CartStore, CartScreen, resumen de pedido |
> | 07 | Checkout y pedidos | Formulario de envío, OrdersStore, historial de pedidos |
> | 08 | Perfil de usuario | ProfileScreen, edición de datos, cambio de contraseña |
> | 09 | Panel Admin — Productos | CRUD de productos, filtros admin, ProductFormModal |
> | 10 | Panel Admin — Pedidos | Lista de todos los pedidos, filtros de estado, cambio de estado |
> | 11 | Panel Admin — Usuarios | CRUD de usuarios, búsqueda con debounce, toggleActive optimista |
> | 12 | Imágenes | FastImage, upload de foto de producto y avatar |
> | 13 | Testing | Jest + RNTL, tests de stores y componentes, cobertura > 70% |
> | 14 | Pulido y UX | Skeletons animados, ErrorBoundary, EmptyState, Toast, FlatList optimizada |
> | 15 | Distribución | Variables de entorno, build Android/iOS, publicación en stores |

---

**Felicitaciones.** Has completado los 15 módulos de la serie ShopApp React Native. La app cuenta con autenticación segura, catálogo con filtros, carrito de compras, panel de administración completo, imágenes con carga optimizada, suite de tests y un proceso de build documentado para distribución en ambas plataformas.
