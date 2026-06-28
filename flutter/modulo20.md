# Tutorial Flutter — Página 20
## Módulo 8 · Publicación
### De desarrollo a producción: APK, AAB, IPA y CI/CD

---

## ¿Qué aprenderás?

```
Modo desarrollo                      Modo producción / release
──────────────────────────────────   ──────────────────────────────────────
flutter run (debug)                  flutter build appbundle --release
Sin firma                            Keystore firmado con keytool
APK de desarrollo (grande)           AAB optimizado (Google lo divide)
Hot reload activo                    Código minificado + obfuscado
debugShowCheckedModeBanner: true     debugShowCheckedModeBanner: false
Logs en consola                      Sin logs de debug
Versión 1.0.0+1                      Incrementar: 1.0.1+2, 1.1.0+3...
```

El pipeline completo de publicación luce así:

```
Código → flutter analyze → flutter test → flutter build appbundle --release
                                                        ↓
                                             Google Play Console
                                      Internal → Alpha → Beta → Producción
```

**Importante:** en este módulo no hay una app que modificar en tiempo real.
En cambio, ejecutarás comandos en la terminal, editarás archivos de configuración
y crearás un workflow de CI/CD. El proyecto base es `productos_clean` de la página 19.

---

## Patrones clave

Los 6 patrones siguientes cubren los conceptos esenciales de publicación.
Cada bloque sigue el ritmo: **concepto puro → ejemplo de comando/config → mini-ejercicio**.

---

### Patrón 1 — Versión semántica en pubspec.yaml

El campo `version` de `pubspec.yaml` controla dos números que Flutter convierte
en constantes nativas:

```yaml
# pubspec.yaml
version: 1.2.3+45
#        │─┬─│ ││
#        │ │ │ └┴─ build number — entero que Google Play usa como versionCode
#        │ │ └──── patch  — correcciones de bugs (1.2.3 → 1.2.4)
#        │ └────── minor  — nuevas features sin romper compatibilidad (1.2.x → 1.3.0)
#        └──────── major  — cambios incompatibles o rediseño (1.x.x → 2.0.0)
```

Reglas prácticas:
- El **build number** debe incrementarse en **cada** subida a la tienda, sin excepción.
- El **versionName** (`major.minor.patch`) se muestra al usuario.
- Google Play rechaza un AAB cuyo `versionCode` sea igual o menor al ya publicado.

```bash
# Ver la versión actual sin abrir el archivo
grep '^version:' pubspec.yaml

# Antes de publicar, editar pubspec.yaml manualmente:
# version: 1.0.0+1  →  version: 1.0.0+2

# También puedes usar el paquete pub_semver para automatizarlo (ver Ejercicio 1)
```

> **Mini-ejercicio ⏱ 5 min**
>
> 1. Abre `pubspec.yaml` de `productos_clean`.
> 2. Cambia la versión de `1.0.0+1` a `1.0.0+2`.
> 3. Ejecuta `grep '^version:' pubspec.yaml` y confirma el cambio.
> 4. Discute con tu compañero: ¿cuándo incrementarías el número **major**?
>    ¿Cuándo solo el **build number**?

---

### Patrón 2 — Íconos automáticos con flutter_launcher_icons

Generar íconos para todas las resoluciones de Android e iOS a mano es tedioso.
El paquete `flutter_launcher_icons` lo hace con un solo comando.

```yaml
# pubspec.yaml — sección completa con el paquete y su configuración

name: productos_clean
description: App de gestión de productos.
version: 1.0.0+2
publish_to: 'none'

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  # ... otras dependencias

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_launcher_icons: ^0.14.1   # <── agregar aquí

# Configuración de flutter_launcher_icons al final del archivo
flutter_icons:
  android: true
  ios:     true
  image_path: "assets/icons/icon.png"       # imagen fuente 1024×1024 px
  adaptive_icon_background: "#FFFFFF"        # fondo para Android 8+ (Oreo+)
  adaptive_icon_foreground: "assets/icons/icon_foreground.png"
  # web:
  #   generate: true
  #   image_path: "assets/icons/icon.png"
```

```bash
# 1. Instalar el paquete
flutter pub get

# 2. Crear la carpeta de assets si no existe
mkdir -p assets/icons

# 3. Coloca tu imagen icon.png (mínimo 512×512 px, recomendado 1024×1024)
#    en assets/icons/icon.png

# 4. Generar íconos para todas las resoluciones
flutter pub run flutter_launcher_icons

# Salida esperada:
#  • Generating icons for Android...
#  • Generating icons for iOS...
#  ✓ Successfully generated launcher icons
```

Después de ejecutarlo, los íconos aparecen en:
- Android: `android/app/src/main/res/mipmap-*/`
- iOS: `ios/Runner/Assets.xcassets/AppIcon.appiconset/`

> **Mini-ejercicio ⏱ 6 min**
>
> 1. Agrega `flutter_launcher_icons: ^0.14.1` a `dev_dependencies`.
> 2. Agrega la sección `flutter_icons` al final de `pubspec.yaml`.
> 3. Crea `assets/icons/` y coloca cualquier PNG de 512×512 px o mayor.
>    (Puedes descargar uno gratuito de flaticon.com o usar uno propio).
> 4. Ejecuta `flutter pub run flutter_launcher_icons`.
> 5. Verifica que `android/app/src/main/res/mipmap-hdpi/ic_launcher.png` existe.

---

### Patrón 3 — Generación del keystore (firma Android)

Toda app de Android debe estar **firmada digitalmente** antes de publicarse.
El keystore es un archivo `.jks` que contiene tu clave privada.

**Regla de oro:** si pierdes el keystore, no puedes publicar actualizaciones de tu app.
Guárdalo en al menos dos lugares seguros y **nunca lo pongas en git**.

```bash
# Generar keystore — ejecutar UNA sola vez en tu vida para cada app
# (si ya tienes uno, reutilízalo para todas las actualizaciones)
keytool -genkey -v \
  -keystore ~/upload-keystore.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias upload

# El comando pedirá de forma interactiva:
# Enter keystore password:  [tu contraseña — escríbela dos veces]
# Re-enter new password:
# What is your first and last name? [Juan Pérez]
# What is the name of your organizational unit? [Desarrollo]
# What is the name of your organization?  [Mi Empresa]
# What is the name of your City or Locality? [Bogotá]
# What is the name of your State or Province? [Cundinamarca]
# What is the two-letter country code? [CO]
# Is CN=Juan Pérez, OU=Desarrollo, O=Mi Empresa, L=Bogotá, ST=Cundinamarca, C=CO correct?
# [no]:  yes
```

Con el keystore generado, crea el archivo de propiedades que Gradle leerá:

```properties
# android/key.properties
# IMPORTANTE: este archivo va en .gitignore — nunca en el repositorio
storePassword=tu_contraseña_store
keyPassword=tu_contraseña_key
keyAlias=upload
storeFile=/home/tu_usuario/upload-keystore.jks
```

Agrega al `.gitignore` de Android:

```
# android/.gitignore
key.properties
**/*.jks
**/*.keystore
```

> **Mini-ejercicio ⏱ 7 min**
>
> 1. Ejecuta el comando `keytool` anterior para generar `~/upload-keystore.jks`.
>    Usa tus datos reales (o inventados para la práctica).
> 2. Crea el archivo `android/key.properties` apuntando a tu keystore.
> 3. Abre `android/.gitignore` y verifica (o agrega) que `key.properties` y `*.jks`
>    están listados.
> 4. Ejecuta `git status` y confirma que `key.properties` no aparece como archivo a trackear.

---

### Patrón 4 — Configurar firma en build.gradle

Gradle necesita leer el keystore al momento de compilar el APK/AAB de release.
La configuración se hace en `android/app/build.gradle`.

```groovy
// android/app/build.gradle — archivo completo con firma configurada

// ── Leer key.properties ANTES del bloque android {} ────────────────
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

android {
    compileSdkVersion flutter.compileSdkVersion
    ndkVersion flutter.ndkVersion

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    defaultConfig {
        // CAMBIAR antes de publicar — debe ser único en todo Google Play
        applicationId "com.tuempresa.productos"
        minSdkVersion 21
        targetSdkVersion 34
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
    }

    // ── Configuración de firma ───────────────────────────────────
    signingConfigs {
        release {
            keyAlias      keystoreProperties['keyAlias']
            keyPassword   keystoreProperties['keyPassword']
            storeFile     keystoreProperties['storeFile'] ?
                          file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            signingConfig  signingConfigs.release  // usar la firma definida arriba
            minifyEnabled  true                    // minificar código Java/Kotlin
            shrinkResources true                   // eliminar recursos no usados
        }
    }
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
}

apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"
```

> **Mini-ejercicio ⏱ 6 min**
>
> 1. Abre `android/app/build.gradle` de `productos_clean`.
> 2. Agrega el bloque de lectura de `keystoreProperties` antes del bloque `android {}`.
> 3. Agrega `signingConfigs.release` dentro del bloque `android {}`.
> 4. Cambia `applicationId` a algo con tu nombre o empresa (ej: `com.miempresa.productos`).
> 5. Ejecuta `flutter build apk --release`.
>    Si el resultado termina en `BUILD SUCCESSFUL`, la firma está correcta.

**Verifica:** el APK firmado debe aparecer en:
```
build/app/outputs/flutter-apk/app-release.apk
```

---

### Patrón 5 — Comandos de build

Con el keystore y Gradle configurados, los comandos de build son directos.

```bash
# ── Android ────────────────────────────────────────────────────────────────

# APK universal — un solo archivo, funciona en todos los dispositivos
# Más grande (~15-30 MB extra). Útil para distribución directa (no Play Store)
flutter build apk --release

# APK dividido por ABI — un APK por arquitectura de CPU (más pequeño cada uno)
# arm64-v8a (la mayoría de dispositivos modernos), armeabi-v7a, x86_64
flutter build apk --split-per-abi --release

# AAB — Android App Bundle (formato requerido por Google Play desde 2021)
# Google genera APKs optimizados para cada dispositivo a partir del AAB
flutter build appbundle --release

# AAB con obfuscación — dificulta la ingeniería inversa del código Dart
# Requiere guardar los archivos de debug-info para descramblar stack traces
flutter build appbundle --release \
  --obfuscate \
  --split-debug-info=build/debug-info/

# ── iOS (solo desde macOS con Xcode instalado) ─────────────────────────────
flutter build ipa --release

# ── Verificar los artefactos generados ────────────────────────────────────
# APK universal
ls -lh build/app/outputs/flutter-apk/app-release.apk

# APKs por ABI
ls -lh build/app/outputs/flutter-apk/

# AAB
ls -lh build/app/outputs/bundle/release/app-release.aab

# Instalar APK directamente en un dispositivo Android conectado por USB
flutter install --release

# Verificar con adb que la app arranca
adb shell am start -n com.tuempresa.productos/.MainActivity
```

> **Mini-ejercicio ⏱ 7 min**
>
> 1. Ejecuta `flutter build apk --split-per-abi --release`.
> 2. Ejecuta `ls -lh build/app/outputs/flutter-apk/` y anota el tamaño de los
>    tres APKs generados: `arm64-v8a`, `armeabi-v7a` y `x86_64`.
> 3. ¿Cuál es el más grande? ¿Por qué el APK universal pesa más que cualquiera
>    de los tres individuales?
> 4. Si tienes un dispositivo Android, conecta por USB y ejecuta
>    `flutter install --release` para instalar el APK de release.

**Verifica:** al abrir la app instalada en modo release notarás que:
- No hay el banner rojo "DEBUG" en la esquina superior derecha.
- Los logs de `print()` ya no aparecen en la terminal.
- La app es notablemente más fluida que en modo debug.

---

### Patrón 6 — Workflow GitHub Actions (CI/CD)

GitHub Actions automatiza el ciclo: cada push a `main` dispara análisis,
tests y construcción del AAB. El resultado queda como artefacto descargable.

```yaml
# .github/workflows/flutter_ci.yml
name: Flutter CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:          # permite correr manualmente desde la UI de GitHub
    inputs:
      version:
        description: 'Versión a publicar (ej: 1.0.1+2)'
        required: false
        default: ''

jobs:
  # ── Job 1: Análisis estático y tests ──────────────────────────────────────
  test:
    name: Análisis y Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.x'
          channel: stable

      - name: Instalar dependencias
        run: flutter pub get

      - name: Análisis estático
        run: flutter analyze

      - name: Ejecutar tests
        run: flutter test

  # ── Job 2: Build del AAB (solo si los tests pasan) ────────────────────────
  build-android:
    name: Build Android (AAB)
    needs: test                  # este job espera que 'test' termine sin errores
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.x'
          channel: stable

      - name: Instalar dependencias
        run: flutter pub get

      # Decodificar el keystore desde el secret de GitHub (guardado en base64)
      - name: Configurar firma Android
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d \
            > android/app/upload-keystore.jks

          # Crear key.properties apuntando al keystore decodificado
          cat > android/key.properties <<EOF
          storePassword=${{ secrets.STORE_PASSWORD }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          storeFile=upload-keystore.jks
          EOF

      - name: Build AAB
        run: flutter build appbundle --release

      # Guardar el AAB como artefacto descargable durante 30 días
      - uses: actions/upload-artifact@v4
        with:
          name: app-release-aab
          path: build/app/outputs/bundle/release/app-release.aab
          retention-days: 30
```

Para que el workflow pueda firmar el AAB, configura los secrets en GitHub:

```bash
# 1. Codificar el keystore en base64 (en tu máquina local)
base64 -w 0 ~/upload-keystore.jks
# Copia toda la salida (una línea muy larga de caracteres)

# 2. En GitHub → tu repositorio → Settings → Secrets and variables → Actions
#    → New repository secret — agrega los cuatro secrets:
#
#    KEYSTORE_BASE64  = la salida del comando base64 anterior
#    STORE_PASSWORD   = contraseña del keystore (la que usaste en keytool)
#    KEY_PASSWORD     = contraseña de la key (normalmente igual a STORE_PASSWORD)
#    KEY_ALIAS        = upload
```

> **Mini-ejercicio ⏱ 8 min**
>
> 1. Crea la carpeta `.github/workflows/` en la raíz del proyecto.
> 2. Crea `.github/workflows/flutter_ci.yml` con **solo el job `test`**
>    (omite el job `build-android` por ahora).
> 3. Haz commit y push a `main`.
> 4. Ve a la pestaña **Actions** de tu repositorio en GitHub y verifica que
>    el workflow aparece y corre `flutter analyze` y `flutter test`.
> 5. ¿El workflow pasó? Si falló, lee el log de error y corrígelo.

---

## Proyecto base

Este módulo usa `productos_clean` creado en la página 19.
Si no lo tienes, puedes crearlo desde cero:

```bash
# Opción A — clonar desde la página 19 si está en git
git clone https://github.com/tu-usuario/productos_clean.git
cd productos_clean
flutter pub get

# Opción B — crear proyecto nuevo para practicar este módulo
flutter create productos_clean --org com.tuempresa
cd productos_clean
flutter pub get
flutter run   # verificar que arranca en modo debug antes de configurar release
```

Estructura mínima esperada antes de comenzar este módulo:

```
productos_clean/
├── android/
│   ├── app/
│   │   ├── build.gradle        ← editaremos este archivo
│   │   └── src/main/
│   │       └── AndroidManifest.xml
│   ├── .gitignore              ← agregaremos key.properties aquí
│   └── build.gradle
├── ios/
│   └── Runner/
│       └── Info.plist
├── lib/
│   └── main.dart               ← cambiaremos debugShowCheckedModeBanner
├── test/
│   └── widget_test.dart
├── pubspec.yaml                ← editaremos versión e íconos
├── .gitignore                  ← agregaremos *.jks
└── .github/
    └── workflows/
        └── flutter_ci.yml      ← crearemos este archivo
```

---

## Paso 1 — Preparar el proyecto para release

Antes de cualquier build de producción, el proyecto debe pasar un checklist mínimo.

### 1.1 — Limpiar warnings y pasar los tests

```bash
# Análisis estático — debe terminar sin errores ni warnings
flutter analyze

# Salida esperada (ningún issue):
#   Analyzing productos_clean...
#   No issues found! (ran in 2.3s)

# Ejecutar todos los tests
flutter test

# Salida esperada:
#   00:01 +3: All tests passed!
```

Si `flutter analyze` muestra warnings, corrígelos antes de continuar.
Los warnings no bloquean el build, pero son deuda técnica.

### 1.2 — Actualizar la versión

```yaml
# pubspec.yaml
name: productos_clean
version: 1.0.0+1    # ← cambiar el build number antes de cada subida a la tienda
```

### 1.3 — Verificar el nombre de la app

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<application
    android:label="Productos"      <!-- ← nombre que ve el usuario en el launcher -->
    android:icon="@mipmap/ic_launcher"
    ...>
```

```xml
<!-- ios/Runner/Info.plist -->
<key>CFBundleDisplayName</key>
<string>Productos</string>         <!-- ← nombre que ve el usuario en iOS -->
```

### 1.4 — Deshabilitar el banner de debug

```dart
// lib/main.dart
void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Productos',
      debugShowCheckedModeBanner: false,   // ← false en producción
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const HomePage(),
    );
  }
}
```

### 1.5 — Probar en modo release antes del build final

```bash
# Ejecuta la app en modo release en un dispositivo conectado
# Sin hot reload, con el rendimiento real de producción
flutter run --release
```

**Verifica:**
- La app arranca sin el banner rojo "DEBUG" en la esquina.
- El rendimiento es notablemente más fluido (sin el overhead de debug).
- No aparecen prints de debug en la terminal (están optimizados en release).

---

## Paso 2 — Firma Android (keystore)

La firma digital garantiza que las actualizaciones de tu app vienen del mismo autor.
Google Play verifica la firma en cada subida; si no coincide, rechaza el AAB.

### 2.1 — Generar el keystore

```bash
keytool -genkey -v \
  -keystore ~/upload-keystore.jks \
  -keyalg RSA \
  -keysize 2048 \
  -validity 10000 \
  -alias upload

# Duración de validez: 10000 días ≈ 27 años
# RSA 2048 bits es el estándar actual requerido por Google Play
```

### 2.2 — Crear key.properties

```properties
# android/key.properties
# NUNCA commitear este archivo — contiene contraseñas en texto plano
storePassword=mi_contraseña_segura
keyPassword=mi_contraseña_segura
keyAlias=upload
storeFile=/home/tu_usuario/upload-keystore.jks
# En Windows: storeFile=C:\\Users\\tu_usuario\\upload-keystore.jks
```

### 2.3 — Actualizar .gitignore

```
# .gitignore (raíz del proyecto)
# Flutter
.dart_tool/
build/

# Android — secrets
*.jks
*.keystore
android/key.properties

# Firebase (si lo usas)
**/google-services.json
**/GoogleService-Info.plist
```

### 2.4 — Configurar build.gradle

```groovy
// android/app/build.gradle — agregar antes del bloque android {}
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}
```

Dentro del bloque `android {}`, después de `defaultConfig {}`:

```groovy
signingConfigs {
    release {
        keyAlias      keystoreProperties['keyAlias']
        keyPassword   keystoreProperties['keyPassword']
        storeFile     keystoreProperties['storeFile'] ?
                      file(keystoreProperties['storeFile']) : null
        storePassword keystoreProperties['storePassword']
    }
}

buildTypes {
    release {
        signingConfig   signingConfigs.release
        minifyEnabled   true
        shrinkResources true
    }
}
```

**Verifica:**

```bash
flutter build apk --release
# Salida: BUILD SUCCESSFUL in Xs
# Archivo: build/app/outputs/flutter-apk/app-release.apk

# Verificar que el APK está firmado
# (requiere build-tools instalado en el Android SDK)
~/.local/share/android-sdk/build-tools/34.0.0/apksigner verify \
  build/app/outputs/flutter-apk/app-release.apk
# Salida esperada: Verifies
```

---

## Paso 3 — Build Android (APK y AAB)

Con la firma configurada, el build de release es un solo comando.

### 3.1 — Build del AAB (formato para Google Play)

```bash
# AAB estándar — requerido por Google Play
flutter build appbundle --release

# AAB con obfuscación (recomendado para apps en producción)
flutter build appbundle --release \
  --obfuscate \
  --split-debug-info=build/debug-info/

# La carpeta debug-info contiene mapas para descramblar stack traces en Crashlytics
# Guarda esta carpeta junto con cada versión publicada
```

### 3.2 — APKs para distribución directa o pruebas

```bash
# APK universal (un archivo, más grande)
flutter build apk --release

# APKs divididos por arquitectura (más pequeños, para pruebas)
flutter build apk --split-per-abi --release

# Los tres APKs se generan en:
# build/app/outputs/flutter-apk/app-arm64-v8a-release.apk   (dispositivos modernos)
# build/app/outputs/flutter-apk/app-armeabi-v7a-release.apk (dispositivos antiguos)
# build/app/outputs/flutter-apk/app-x86_64-release.apk      (emuladores)
```

### 3.3 — Cambiar el applicationId antes de publicar

El `applicationId` es el identificador único de tu app en Google Play.
**No puede cambiarse después de publicar.** Elige bien.

```groovy
// android/app/build.gradle
defaultConfig {
    applicationId "com.tuempresa.productos"   // ← cambiar ANTES de la primera subida
    minSdkVersion 21           // Android 5.0 — cubre >95% de dispositivos activos
    targetSdkVersion 34        // Android 14 — requerido por Google Play desde 2024
    versionCode    flutterVersionCode.toInteger()
    versionName    flutterVersionName
}
```

### 3.4 — Instalar en dispositivo para prueba final

```bash
# Con dispositivo Android conectado por USB (depuración USB habilitada)
flutter install --release

# Verificar con adb
adb devices           # listar dispositivos conectados
adb shell am start -n com.tuempresa.productos/.MainActivity
```

**Verifica:**
```
build/app/outputs/bundle/release/app-release.aab   ← subir este a Google Play
build/app/outputs/flutter-apk/app-release.apk      ← para pruebas directas
```

### 3.5 — Subir a Google Play Console

```
Google Play Console  →  https://play.google.com/console
Pasos:
1. Crear cuenta de desarrollador ($25 pago único)
2. Crear nueva app → nombre, idioma, tipo (app/juego), gratis/pago
3. Ir a: Pruebas → Pruebas internas → Crear nueva versión
4. Subir el archivo app-release.aab
5. Agregar novedades de la versión en español
6. Guardar y publicar en pruebas internas
7. Compartir el enlace de prueba con testers por email
```

---

## Paso 4 — Build iOS (solo macOS con Xcode)

El build de iOS requiere macOS. Si estás en Linux o Windows, lee esta sección
para entender el proceso pero ejecútala cuando tengas acceso a una Mac.

### 4.1 — Requisitos previos

```
Para publicar en App Store:
- macOS 13 (Ventura) o superior
- Xcode 15 o superior (descarga gratuita desde App Store)
- Apple Developer Program — $99/año (para distribución en App Store)
  Registro: https://developer.apple.com/programs/

Para pruebas en dispositivo físico (sin pagar):
- Solo se requiere una cuenta gratuita de Apple ID
- Limitado a 3 apps simultáneas en dispositivos físicos
```

### 4.2 — Configurar el Bundle ID

El Bundle ID en iOS es equivalente al `applicationId` de Android.
Tampoco puede cambiarse después de publicar.

```
En Xcode → Runner → Target Runner → Bundle Identifier
Ejemplo: com.tuempresa.productos
```

O directamente en el archivo:

```xml
<!-- ios/Runner/Info.plist -->
<key>CFBundleIdentifier</key>
<string>com.tuempresa.productos</string>

<key>CFBundleDisplayName</key>
<string>Productos</string>

<key>CFBundleVersion</key>
<string>$(FLUTTER_BUILD_NUMBER)</string>

<key>CFBundleShortVersionString</key>
<string>$(FLUTTER_VERSION)</string>
```

### 4.3 — Permisos en Info.plist

Apple rechaza apps que solicitan permisos sin justificación en texto.
Agrega solo los permisos que tu app realmente usa:

```xml
<!-- ios/Runner/Info.plist — agregar según necesidad -->

<!-- Cámara -->
<key>NSCameraUsageDescription</key>
<string>Necesitamos acceso a la cámara para tomar fotos de los productos.</string>

<!-- Galería de fotos -->
<key>NSPhotoLibraryUsageDescription</key>
<string>Necesitamos acceso a las fotos para seleccionar imágenes de productos.</string>

<!-- Ubicación (cuando la app está en uso) -->
<key>NSLocationWhenInUseUsageDescription</key>
<string>Usamos tu ubicación para mostrar tiendas cercanas.</string>

<!-- Micrófono -->
<key>NSMicrophoneUsageDescription</key>
<string>Usamos el micrófono para grabar notas de voz.</string>
```

### 4.4 — Comandos de build iOS

```bash
# Build IPA (solo en macOS con Xcode)
flutter build ipa --release

# Salida:
# build/ios/archive/Runner.xcarchive   ← archive completo
# build/ios/ipa/Runner.ipa             ← archivo para TestFlight/App Store

# Abrir el proyecto en Xcode para subir manualmente
open ios/Runner.xcworkspace
# En Xcode: Product → Archive → Distribute App → App Store Connect → Upload
```

### 4.5 — Flujo de publicación en App Store Connect

```
App Store Connect  →  https://appstoreconnect.apple.com
Pasos:
1. Crear nueva app → nombre, Bundle ID, SKU, idioma principal
2. Subir build desde Xcode o con Transporter
3. Completar metadata:
   - Capturas de pantalla (para iPhone 6.7", 6.5", iPad Pro)
   - Descripción corta y larga en español
   - Palabras clave (100 caracteres)
   - URL de política de privacidad (obligatoria)
   - Clasificación de edad
4. TestFlight: pruebas internas (hasta 25 personas) y externas (hasta 10,000)
5. Enviar para revisión de Apple (tiempo promedio: 1-3 días hábiles)
6. Publicar manualmente o de forma automática tras la aprobación
```

**Verifica (en macOS):**
```bash
# Confirmar que el build de iOS terminó
ls -lh build/ios/ipa/Runner.ipa
# Debe existir y pesar varios MB
```

---

## Paso 5 — GitHub Actions: CI/CD completo

Un pipeline de CI/CD bien configurado detecta errores antes de que lleguen a producción
y genera el AAB de forma automática y reproducible, sin depender de la máquina de ningún
desarrollador en particular.

### 5.1 — Crear el directorio del workflow

```bash
mkdir -p .github/workflows
```

### 5.2 — Workflow completo

```yaml
# .github/workflows/flutter_ci.yml
name: Flutter CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      version:
        description: 'Versión a publicar (ej: 1.0.1+2)'
        required: false
        default: ''

jobs:
  # ── Job 1: Análisis y tests ─────────────────────────────────────────────
  test:
    name: Análisis y Tests
    runs-on: ubuntu-latest

    steps:
      - name: Clonar el repositorio
        uses: actions/checkout@v4

      - name: Configurar Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.x'
          channel: stable
          cache: true             # cache de Flutter entre runs (~1 min ahorro)

      - name: Instalar dependencias
        run: flutter pub get

      - name: Análisis estático
        run: flutter analyze --fatal-infos

      - name: Ejecutar tests con coverage
        run: flutter test --coverage

      - name: Subir coverage a Codecov (opcional)
        uses: codecov/codecov-action@v4
        with:
          file: coverage/lcov.info
        continue-on-error: true   # no falla el job si Codecov no está configurado

  # ── Job 2: Build AAB para Android ──────────────────────────────────────
  build-android:
    name: Build Android (AAB)
    needs: test
    runs-on: ubuntu-latest

    steps:
      - name: Clonar el repositorio
        uses: actions/checkout@v4

      - name: Configurar Flutter
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.x'
          channel: stable
          cache: true

      - name: Instalar dependencias
        run: flutter pub get

      - name: Configurar firma Android
        run: |
          # Decodificar el keystore desde el secret
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d \
            > android/app/upload-keystore.jks

          # Crear key.properties para que build.gradle lo lea
          cat > android/key.properties <<EOF
          storePassword=${{ secrets.STORE_PASSWORD }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          storeFile=upload-keystore.jks
          EOF

      - name: Build AAB con obfuscación
        run: |
          flutter build appbundle --release \
            --obfuscate \
            --split-debug-info=build/debug-info/

      - name: Subir AAB como artefacto
        uses: actions/upload-artifact@v4
        with:
          name: app-release-aab-${{ github.run_number }}
          path: build/app/outputs/bundle/release/app-release.aab
          retention-days: 30

      - name: Subir debug-info como artefacto
        uses: actions/upload-artifact@v4
        with:
          name: debug-info-${{ github.run_number }}
          path: build/debug-info/
          retention-days: 90     # guardar más tiempo para stack traces de producción
```

### 5.3 — Configurar secrets en GitHub

```
En tu repositorio GitHub:
  Settings → Secrets and variables → Actions → New repository secret

Secrets necesarios:
┌─────────────────┬──────────────────────────────────────────────────────┐
│ Nombre          │ Valor                                                │
├─────────────────┼──────────────────────────────────────────────────────┤
│ KEYSTORE_BASE64 │ Salida de: base64 -w 0 ~/upload-keystore.jks        │
│ STORE_PASSWORD  │ La contraseña que elegiste en keytool                │
│ KEY_PASSWORD    │ La contraseña de la key (normalmente igual)          │
│ KEY_ALIAS       │ upload                                               │
└─────────────────┴──────────────────────────────────────────────────────┘
```

```bash
# Comando para obtener el valor de KEYSTORE_BASE64
base64 -w 0 ~/upload-keystore.jks
# Copia toda la salida (una sola línea, sin espacios ni saltos de línea)
```

### 5.4 — Descargar el artefacto AAB

```
Después de que el workflow termina:
1. Ve a la pestaña Actions de tu repositorio
2. Haz clic en el run más reciente
3. En la sección "Artifacts" (abajo de la página del run):
   app-release-aab-N  →  Download
4. Descomprime el ZIP descargado
5. Sube el app-release.aab a Google Play Console
```

**Verifica:**
- El job `test` aparece en verde antes de que `build-android` comience.
- El workflow completo tarda aproximadamente 4-8 minutos en un runner `ubuntu-latest`.
- El artefacto `app-release-aab-N` aparece al final del run y es descargable.

---

## Checklist de publicación — referencia completa

### Android — antes de cada subida a Google Play

#### Configuración del proyecto
- [ ] `pubspec.yaml`: versión incrementada (`major.minor.patch+buildNumber`)
- [ ] `pubspec.yaml`: nombre del paquete correcto (no `example` en el nombre)
- [ ] `applicationId` único en `android/app/build.gradle` (no `com.example.*`)
- [ ] `android:label` con nombre real en `AndroidManifest.xml`
- [ ] Ícono de app generado con `flutter_launcher_icons` (fuente 1024×1024 px)
- [ ] `debugShowCheckedModeBanner: false` en `MaterialApp`

#### Calidad del código
- [ ] `flutter analyze` — sin errores ni warnings
- [ ] `flutter test` — todos los tests pasan
- [ ] `flutter run --release` — app probada en modo release en dispositivo real

#### Firma
- [ ] Keystore generado y guardado en lugar seguro (no en git)
- [ ] `android/key.properties` configurado con rutas y contraseñas correctas
- [ ] `key.properties` y `*.jks` están en `.gitignore`
- [ ] `build.gradle` configurado con `signingConfigs.release`
- [ ] `flutter build appbundle --release` — BUILD SUCCESSFUL

#### Google Play Console
- [ ] Cuenta de Google Play Developer creada ($25 pago único)
- [ ] App creada en Play Console con nombre y `applicationId` correctos
- [ ] AAB subido a **Internal Testing** (primera versión)
- [ ] Probado con Internal Testing en al menos un dispositivo físico
- [ ] Content rating completado (clasificación de edad)
- [ ] URL de política de privacidad agregada (obligatoria)
- [ ] Capturas de pantalla subidas (mínimo 2 por tipo de dispositivo)
- [ ] Descripción corta (80 caracteres) y larga (4000 caracteres) en español
- [ ] Revisión aprobada por Google (normalmente 1-3 días hábiles)

### iOS — antes de cada subida a App Store

- [ ] Bundle ID único en Xcode (no `com.example.*`)
- [ ] Provisioning profile configurado en Xcode
- [ ] Permisos en `Info.plist` con descripciones en español
- [ ] `flutter build ipa --release` — sin errores
- [ ] App subida desde Xcode: Archive → Distribute App → App Store Connect
- [ ] TestFlight: probado internamente con al menos 2-3 dispositivos
- [ ] App Store Connect: metadata completa (nombre, descripción, capturas)
- [ ] Clasificación de edad completada
- [ ] URL de política de privacidad agregada
- [ ] Enviar para revisión de Apple (1-3 días hábiles promedio)

---

## Proyecto de configuración — archivos completos

### Archivo 1: pubspec.yaml

```yaml
name: productos_clean
description: App de gestión de productos - Tutorial Flutter página 20.
publish_to: 'none'

version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.8
  flutter_riverpod: ^2.5.1
  dio: ^5.4.3
  cached_network_image: ^3.3.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0
  flutter_launcher_icons: ^0.14.1

flutter:
  uses-material-design: true
  assets:
    - assets/icons/
    - assets/images/

# Configuración de flutter_launcher_icons
flutter_icons:
  android: true
  ios:     true
  image_path: "assets/icons/icon.png"
  adaptive_icon_background: "#FFFFFF"
  adaptive_icon_foreground: "assets/icons/icon_foreground.png"
  remove_alpha_ios: true       # App Store rechaza íconos con transparencia
```

### Archivo 2: android/app/build.gradle

```groovy
// android/app/build.gradle

// ── Leer key.properties si existe ────────────────────────────────────────────
def keystoreProperties = new Properties()
def keystorePropertiesFile = rootProject.file('key.properties')
if (keystorePropertiesFile.exists()) {
    keystoreProperties.load(new FileInputStream(keystorePropertiesFile))
}

def localProperties = new Properties()
def localPropertiesFile = rootProject.file('local.properties')
if (localPropertiesFile.exists()) {
    localPropertiesFile.withReader('UTF-8') { reader ->
        localProperties.load(reader)
    }
}

def flutterRoot = localProperties.getProperty('flutter.sdk')
if (flutterRoot == null) {
    throw new GradleException("Flutter SDK not found. Define location with flutter.sdk.")
}

def flutterVersionCode = localProperties.getProperty('flutter.versionCode')
if (flutterVersionCode == null) { flutterVersionCode = '1' }

def flutterVersionName = localProperties.getProperty('flutter.versionName')
if (flutterVersionName == null) { flutterVersionName = '1.0' }

apply plugin: 'com.android.application'
apply plugin: 'kotlin-android'
apply from: "$flutterRoot/packages/flutter_tools/gradle/flutter.gradle"

android {
    namespace "com.tuempresa.productos"
    compileSdkVersion 34
    ndkVersion flutter.ndkVersion

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_1_8
        targetCompatibility JavaVersion.VERSION_1_8
    }

    kotlinOptions {
        jvmTarget = '1.8'
    }

    sourceSets {
        main.java.srcDirs += 'src/main/kotlin'
    }

    defaultConfig {
        applicationId "com.tuempresa.productos"    // ← CAMBIAR antes de publicar
        minSdkVersion 21
        targetSdkVersion 34
        versionCode    flutterVersionCode.toInteger()
        versionName    flutterVersionName
    }

    // ── Firma para release ───────────────────────────────────────────────────
    signingConfigs {
        release {
            keyAlias      keystoreProperties['keyAlias']
            keyPassword   keystoreProperties['keyPassword']
            storeFile     keystoreProperties['storeFile'] ?
                          file(keystoreProperties['storeFile']) : null
            storePassword keystoreProperties['storePassword']
        }
    }

    buildTypes {
        release {
            signingConfig   signingConfigs.release
            minifyEnabled   true      // minifica código Java/Kotlin con R8
            shrinkResources true      // elimina recursos no referenciados
        }
    }
}

flutter {
    source '../..'
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
}
```

### Archivo 3: android/key.properties

```properties
# android/key.properties
# ATENCIÓN: este archivo contiene contraseñas — NUNCA commitear a git
# Agregar "key.properties" a android/.gitignore
storePassword=REEMPLAZAR_CON_TU_CONTRASEÑA
keyPassword=REEMPLAZAR_CON_TU_CONTRASEÑA
keyAlias=upload
storeFile=/home/tu_usuario/upload-keystore.jks
# En Windows usar barras invertidas dobles:
# storeFile=C:\\Users\\tu_usuario\\upload-keystore.jks
```

### Archivo 4: .gitignore (raíz del proyecto)

```gitignore
# Archivos generados por Dart/Flutter
.dart_tool/
.flutter-plugins
.flutter-plugins-dependencies
.packages
build/
pubspec.lock   # dejar sin lockear para apps (sí lockear para paquetes publicados)

# Android — secrets de firma (NUNCA en git)
*.jks
*.keystore
android/key.properties

# Android — archivos locales
android/local.properties
android/.gradle/
android/captures/

# iOS — archivos locales
ios/.symlinks/
ios/Flutter/flutter_export_environment.sh
ios/Flutter/Generated.xcconfig
ios/Pods/

# Firebase
**/google-services.json
**/GoogleService-Info.plist

# IDE
.idea/
.vscode/
*.iml

# macOS
.DS_Store

# Cobertura de tests
coverage/
```

### Archivo 5: .github/workflows/flutter_ci.yml

```yaml
# .github/workflows/flutter_ci.yml
name: Flutter CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      version:
        description: 'Versión a publicar (ej: 1.0.1+2)'
        required: false
        default: ''

jobs:
  test:
    name: Análisis y Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.x'
          channel: stable
          cache: true
      - run: flutter pub get
      - run: flutter analyze --fatal-infos
      - run: flutter test --coverage

  build-android:
    name: Build Android (AAB)
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.x'
          channel: stable
          cache: true
      - run: flutter pub get
      - name: Configurar firma Android
        run: |
          echo "${{ secrets.KEYSTORE_BASE64 }}" | base64 -d \
            > android/app/upload-keystore.jks
          cat > android/key.properties <<EOF
          storePassword=${{ secrets.STORE_PASSWORD }}
          keyPassword=${{ secrets.KEY_PASSWORD }}
          keyAlias=${{ secrets.KEY_ALIAS }}
          storeFile=upload-keystore.jks
          EOF
      - name: Build AAB
        run: |
          flutter build appbundle --release \
            --obfuscate \
            --split-debug-info=build/debug-info/
      - uses: actions/upload-artifact@v4
        with:
          name: app-release-aab-${{ github.run_number }}
          path: build/app/outputs/bundle/release/app-release.aab
          retention-days: 30
      - uses: actions/upload-artifact@v4
        with:
          name: debug-info-${{ github.run_number }}
          path: build/debug-info/
          retention-days: 90
```

### Archivo 6: android/app/src/main/AndroidManifest.xml

```xml
<!-- android/app/src/main/AndroidManifest.xml -->
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- Permisos comunes — agregar solo los que tu app use -->
    <uses-permission android:name="android.permission.INTERNET"/>
    <!-- <uses-permission android:name="android.permission.CAMERA"/> -->
    <!-- <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/> -->
    <!-- <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/> -->

    <application
        android:label="Productos"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:networkSecurityConfig="@xml/network_security_config"
        android:exported="true">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:taskAffinity=""
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">

            <meta-data
                android:name="io.flutter.embedding.android.NormalTheme"
                android:resource="@style/NormalTheme" />

            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <meta-data
            android:name="flutterEmbedding"
            android:value="2" />
    </application>
</manifest>
```

---

## Guía rápida de comandos

| Comando | Qué hace |
|---------|----------|
| `flutter analyze` | Análisis estático del código |
| `flutter test` | Ejecutar todos los tests |
| `flutter run --release` | Probar en release sin publicar en tienda |
| `flutter build apk --release` | APK universal firmado |
| `flutter build apk --split-per-abi --release` | 3 APKs separados por arquitectura de CPU |
| `flutter build appbundle --release` | AAB para Google Play (formato obligatorio) |
| `flutter build ipa --release` | IPA para App Store (solo macOS con Xcode) |
| `flutter install --release` | Instalar APK en dispositivo Android conectado |
| `flutter pub run flutter_launcher_icons` | Generar íconos en todas las resoluciones |
| `keytool -genkey -v -keystore ~/key.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload` | Generar keystore de firma |
| `base64 -w 0 ~/upload-keystore.jks` | Codificar keystore para GitHub Secrets |
| `flutter build appbundle --release --obfuscate --split-debug-info=build/debug-info/` | Build con obfuscación de código |

---

## Cuándo usar qué

```
Formato / Herramienta          Cuándo usarlo                        Nota importante
──────────────────────────     ─────────────────────────────────    ──────────────────────────────────────
APK universal                  Pruebas internas, distribución        Más grande; no recomendado para Play
                               directa fuera de tiendas              Store (Google prefiere AAB)

APK split-per-abi              Pruebas en dispositivos               Genera 3 APKs, uno por arquitectura
                               específicos                           de CPU (arm64, armeabi, x86_64)

AAB (Android App Bundle)       Google Play Store (obligatorio        Google genera APKs optimizados por
                               desde agosto 2021)                    dispositivo a partir del AAB

IPA                            App Store de Apple                    Solo compilable desde macOS con Xcode

flutter run --release          Probar antes de publicar              Sin hot reload; rendimiento real de
                               en dispositivo conectado              producción sin Banner DEBUG

Keystore upload                Firma para Google Play                Google guarda la clave maestra;
                                                                     el upload key puede rotarse si se
                                                                     pierde (Google Play App Signing)

--obfuscate                    Apps con lógica sensible o de        Dificulta ingeniería inversa; siempre
                               negocio privada                       combinar con --split-debug-info

Internal Testing               Primera subida a Play Console         Solo testers con email invitado;
                                                                     tiempo de revisión ~horas

Alpha / Beta                   Pruebas externas antes del           Open o Closed; Alpha = más cerrado,
                               lanzamiento público                   Beta = más abierto

Production                     Lanzamiento público en               Puede ser gradual: 10% → 50% → 100%
                               Google Play Store                     de usuarios antes del rollout completo

GitHub Actions                 CI/CD automatizado en cada           Gratis para repos públicos; 2,000
                               push a main                          min/mes en repos privados (plan Free)

flutter_launcher_icons         Generar íconos automáticamente        Requiere PNG fuente 1024×1024 px;
                               para todas las resoluciones           sin transparencia para iOS
```

---

## Ejercicios propuestos

### Ejercicio 1 — Automatizar el incremento de versión

Crea un script `bump_version.sh` en la raíz del proyecto que:
1. Lea la versión actual de `pubspec.yaml` (p. ej. `1.0.0+1`).
2. Incremente el build number en 1 (p. ej. `1.0.0+1` → `1.0.0+2`).
3. Actualice `pubspec.yaml` con la nueva versión.
4. Muestre la versión anterior y la nueva en la terminal.

```bash
#!/bin/bash
# bump_version.sh — incrementa el build number en pubspec.yaml

# Leer la línea de versión actual
CURRENT=$(grep '^version:' pubspec.yaml | sed 's/version: //')
echo "Versión actual: $CURRENT"

# Separar versionName y buildNumber
VERSION_NAME=$(echo $CURRENT | cut -d'+' -f1)
BUILD_NUMBER=$(echo $CURRENT | cut -d'+' -f2)

# Incrementar build number
NEW_BUILD=$((BUILD_NUMBER + 1))
NEW_VERSION="${VERSION_NAME}+${NEW_BUILD}"

# Actualizar pubspec.yaml
sed -i "s/^version: .*/version: $NEW_VERSION/" pubspec.yaml

echo "Nueva versión:  $NEW_VERSION"
echo "Actualizado en pubspec.yaml"
```

Para ejecutarlo antes de cada build:

```bash
chmod +x bump_version.sh
./bump_version.sh
flutter build appbundle --release
```

**Bonus:** Extiende el script para recibir como argumento el tipo de bump:
`./bump_version.sh patch`, `./bump_version.sh minor`, `./bump_version.sh major`.

---

### Ejercicio 2 — Workflow con notificación al terminar el build

Extiende `.github/workflows/flutter_ci.yml` para que al terminar el job
`build-android` (con éxito o con error) envíe una notificación.

**Opción A — Email con GitHub**
GitHub envía notificaciones automáticas al correo cuando un workflow falla.
Para éxito, agrega un paso al final del job:

```yaml
      - name: Notificar resultado
        if: always()     # corre aunque los pasos anteriores fallen
        run: |
          if [ "${{ job.status }}" == "success" ]; then
            echo "BUILD EXITOSO — AAB disponible en la pestaña Actions"
          else
            echo "BUILD FALLIDO — revisar logs en Actions"
          fi
```

**Opción B — Slack (si tienes un workspace)**

```yaml
      - name: Notificar en Slack
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: |
            Build Android ${{ job.status == 'success' && 'exitoso' || 'fallido' }}
            Repositorio: ${{ github.repository }}
            Rama: ${{ github.ref_name }}
            Run: ${{ github.run_number }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

### Ejercicio 3 — Fake Flavors para desarrollo y producción

Configura dos variantes de compilación (flavors) en Android sin usar el
plugin oficial de flavors de Flutter, usando solo variables en `build.gradle`:

```groovy
// android/app/build.gradle — dentro de android {}
flavorDimensions "environment"

productFlavors {
    development {
        dimension     "environment"
        applicationId "com.tuempresa.productos.dev"
        versionNameSuffix "-dev"
        resValue "string", "app_name", "Productos DEV"
    }
    production {
        dimension     "environment"
        applicationId "com.tuempresa.productos"
        resValue "string", "app_name", "Productos"
    }
}
```

En `AndroidManifest.xml`, usa el valor dinámico:

```xml
<application android:label="@string/app_name" ...>
```

Scripts para ejecutar cada flavor:

```bash
# run_dev.sh — modo desarrollo
flutter run --flavor development -t lib/main_dev.dart

# build_prod.sh — build de producción
flutter build appbundle --release --flavor production -t lib/main.dart
```

En `lib/main_dev.dart`, muestra un banner visible en la AppBar:

```dart
// lib/main_dev.dart
import 'package:flutter/material.dart';
import 'main.dart' as app;

void main() {
  // Configuración especial para desarrollo
  app.main();
}
// El flavor DEV también puede activarse en main.dart con una constante:
// const bool isDevFlavor = String.fromEnvironment('FLAVOR') == 'development';
```

**Bonus:** Haz que el flavor `development` muestre "DEV" en la AppBar
usando `const String.fromEnvironment('FLAVOR')`.

---

### Ejercicio 4 — Análisis de tamaño del AAB

El tamaño del AAB afecta directamente la tasa de instalación de tu app.
Google rechaza apps mayores a 150 MB (en AAB) para la mayoría de los casos.

```bash
# Build con análisis de tamaño habilitado
flutter build appbundle --analyze-size

# Salida de ejemplo:
#
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# app-release.aab (total compressed)                         8.0 MB
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# dart (AOT precompiled)
#   Dart AOT symbols accounted decompressed size             2.2 MB
#   package:flutter                                          1.1 MB
#   package:dio                                              0.3 MB
#   package:cached_network_image                             0.2 MB
#   dart:core                                                0.1 MB
# resources                                                  1.5 MB
# assets                                                     0.8 MB
# native code                                                3.5 MB
```

**Pasos del ejercicio:**

1. Ejecuta `flutter build appbundle --analyze-size` en `productos_clean`.
2. Identifica el paquete que ocupa más espacio en la sección `dart`.
3. Busca en pub.dev si existe una alternativa más ligera para ese paquete.
4. Si el paquete `cached_network_image` aparece, compara su tamaño con
   `flutter_cache_manager` directamente.
5. Anota el tamaño total del AAB antes y después de cualquier optimización.

**Reducir el tamaño:**

```bash
# Eliminar paquetes no usados
flutter pub deps | grep -E "^[a-z]"   # listar dependencias directas

# Revisar qué iconos de Material Icons se usan realmente
# El tree shaking de Flutter elimina los no usados automáticamente en release

# Si usas imágenes PNG grandes, considera convertirlas a WebP
# (mejor compresión sin pérdida de calidad visible)
```

---

## Resumen

- El flujo de publicación Android es: `flutter analyze` → `flutter test` →
  `flutter build appbundle --release` → subir AAB a Google Play Console.
- El **AAB** (Android App Bundle) es el formato obligatorio para Google Play
  desde agosto de 2021; Google genera APKs optimizados por dispositivo a partir de él.
- El **keystore** es irreemplazable: si lo pierdes, no puedes publicar actualizaciones
  de tu app en Google Play. Guárdalo en mínimo dos lugares seguros y nunca lo
  pongas en git.
- `key.properties` va en `.gitignore`; en CI/CD los secrets se inyectan como
  variables de entorno en GitHub Actions, codificando el keystore en base64.
- En GitHub Actions: el keystore se codifica con `base64 -w 0 ~/key.jks`,
  se guarda como secret `KEYSTORE_BASE64`, y se decodifica en el workflow
  antes del paso de build.
- `--obfuscate` combinado con `--split-debug-info` dificulta la ingeniería
  inversa del código Dart y es una práctica recomendada para apps en producción.
- El `applicationId` en Android y el `Bundle ID` en iOS deben ser únicos
  globalmente y **no pueden cambiarse después de la primera publicación**.
- Para iOS se necesita macOS + Xcode + Apple Developer Program ($99/año);
  para Android basta con cualquier SO y un pago único de $25 en Google Play Console.

---

> **Fin del curso Flutter** Has completado las 20 páginas del Tutorial Flutter.
>
> Siguiente recomendación: practica con un proyecto propio completo combinando
> lo aprendido en las páginas 1-20. Revisa también los cursos complementarios:
> **React.js + TypeScript**, **NestJS Unit Tests** y **Kotlin**.
