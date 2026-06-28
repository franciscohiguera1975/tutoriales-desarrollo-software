# ShopApp Android — Módulo 1

## Setup + Estructura + Design System Material 3 + Tipos de datos

> **Objetivo:** Proyecto Android con Kotlin y Jetpack Compose configurado, estructura de carpetas profesional, design system con Material 3 y modelos de dominio definidos.
> **Checkpoint final:** la app arranca en el emulador mostrando la pantalla de verificación con el tema aplicado.

---

## 1.1 Crear el proyecto

En **Android Studio**:

1. **File → New → New Project**
2. Seleccionar **Empty Activity** con Jetpack Compose
3. Configurar:

   * **Name:** ShopApp
   * **Package name:** `com.shopapp`
   * **Save location:** tu ruta preferida
   * **Language:** Kotlin
   * **Minimum SDK:** API 26 Android 8.0
4. Clic en **Finish**

---

## 1.2 Configuración base de Gradle

Antes de agregar dependencias, es importante dejar Gradle configurado correctamente.

> **Importante:** en este proyecto se usará la forma moderna con `plugins` y `libs.versions.toml`.
> No debes usar `buildscript { classpath(...) }`, porque eso puede generar errores como:

```text
The request for this plugin could not be satisfied because the plugin is already on the classpath with an unknown version
```

También puede aparecer este error si se deja una línea incompleta como `kotlin-gradle-plugin:...`:

```text
Cannot resolve external dependency org.jetbrains.kotlin:kotlin-gradle-plugin:... because no repositories are defined
```

Por eso, los archivos deben quedar como se muestra a continuación.

---

## 1.3 `settings.gradle.kts`

Reemplaza el contenido del archivo `settings.gradle.kts` de la raíz del proyecto por este:

```kotlin
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

rootProject.name = "ShopApp"
include(":app")
```

---

## 1.4 Version Catalog — `gradle/libs.versions.toml`

Reemplaza el contenido de `gradle/libs.versions.toml` por este:

```toml
[versions]
agp = "8.7.2"
kotlin = "2.0.21"
coreKtx = "1.13.1"
lifecycleRuntimeKtx = "2.8.6"
activityCompose = "1.9.3"
lifecycleViewmodelCompose = "2.8.6"
navigationCompose = "2.8.3"
composeBom = "2024.10.01"
hilt = "2.52"
hiltNavigationCompose = "1.2.0"
retrofit = "2.11.0"
okhttp = "4.12.0"
datastore = "1.1.1"
coroutines = "1.8.1"
serializationJson = "1.7.3"
coil = "2.7.0"

[libraries]
androidx-core-ktx = { group = "androidx.core", name = "core-ktx", version.ref = "coreKtx" }
androidx-lifecycle-runtime-ktx = { group = "androidx.lifecycle", name = "lifecycle-runtime-ktx", version.ref = "lifecycleRuntimeKtx" }
androidx-lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycleViewmodelCompose" }
androidx-activity-compose = { group = "androidx.activity", name = "activity-compose", version.ref = "activityCompose" }
androidx-navigation-compose = { group = "androidx.navigation", name = "navigation-compose", version.ref = "navigationCompose" }

androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
androidx-compose-ui = { group = "androidx.compose.ui", name = "ui" }
androidx-compose-ui-tooling-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
androidx-compose-ui-tooling = { group = "androidx.compose.ui", name = "ui-tooling" }
androidx-compose-material3 = { group = "androidx.compose.material3", name = "material3" }
androidx-compose-material-icons-extended = { group = "androidx.compose.material", name = "material-icons-extended" }

hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }
androidx-hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version.ref = "hiltNavigationCompose" }

retrofit = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-converter-gson = { group = "com.squareup.retrofit2", name = "converter-gson", version.ref = "retrofit" }
okhttp = { group = "com.squareup.okhttp3", name = "okhttp", version.ref = "okhttp" }
okhttp-logging-interceptor = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }

androidx-datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "serializationJson" }
coil-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
hilt-android = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
kotlin-kapt = { id = "org.jetbrains.kotlin.kapt", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
```

---

## 1.5 `build.gradle.kts` raíz del proyecto

Reemplaza el contenido del archivo `build.gradle.kts` de la raíz por este:

```kotlin
plugins {
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.kotlin.compose) apply false
    alias(libs.plugins.hilt.android) apply false
    alias(libs.plugins.kotlin.kapt) apply false
    alias(libs.plugins.kotlin.serialization) apply false
}
```

> **No agregues este bloque:**

```kotlin
buildscript {
    dependencies {
        classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:...")
    }
}
```

Ese bloque no se usa en este tutorial.

---

## 1.6 Dependencias — `app/build.gradle.kts`

Reemplaza el contenido de `app/build.gradle.kts` por este:

```kotlin
import java.util.Properties

plugins {
    alias(libs.plugins.android.application)
    alias(libs.plugins.kotlin.android)
    alias(libs.plugins.kotlin.compose)
    alias(libs.plugins.hilt.android)
    alias(libs.plugins.kotlin.kapt)
    alias(libs.plugins.kotlin.serialization)
}

val localProperties = Properties()
val localPropertiesFile = rootProject.file("local.properties")

if (localPropertiesFile.exists()) {
    localProperties.load(localPropertiesFile.inputStream())
}

val apiBaseUrl = localProperties.getProperty(
    "API_BASE_URL",
    "http://10.0.2.2:8000/api/"
)

android {
    namespace = "com.shopapp"
    compileSdk = 35

    defaultConfig {
        applicationId = "com.shopapp"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0"

        buildConfigField(
            "String",
            "API_BASE_URL",
            "\"$apiBaseUrl\""
        )
    }

    buildFeatures {
        compose = true
        buildConfig = true
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }

    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    // ── Compose BOM ───────────────────────────────────────
    implementation(platform(libs.androidx.compose.bom))
    implementation(libs.androidx.compose.ui)
    implementation(libs.androidx.compose.ui.tooling.preview)
    implementation(libs.androidx.compose.material3)
    implementation(libs.androidx.compose.material.icons.extended)
    debugImplementation(libs.androidx.compose.ui.tooling)

    // ── Core Android ──────────────────────────────────────
    implementation(libs.androidx.core.ktx)
    implementation(libs.androidx.lifecycle.runtime.ktx)
    implementation(libs.androidx.lifecycle.viewmodel.compose)
    implementation(libs.androidx.activity.compose)

    // ── Navegación ────────────────────────────────────────
    implementation(libs.androidx.navigation.compose)

    // ── Hilt DI ───────────────────────────────────────────
    implementation(libs.hilt.android)
    kapt(libs.hilt.compiler)
    implementation(libs.androidx.hilt.navigation.compose)

    // ── Retrofit + OkHttp ─────────────────────────────────
    implementation(libs.retrofit)
    implementation(libs.retrofit.converter.gson)
    implementation(libs.okhttp)
    implementation(libs.okhttp.logging.interceptor)

    // ── DataStore ─────────────────────────────────────────
    implementation(libs.androidx.datastore.preferences)

    // ── Coroutines ────────────────────────────────────────
    implementation(libs.kotlinx.coroutines.android)

    // ── Serialización ─────────────────────────────────────
    implementation(libs.kotlinx.serialization.json)

    // ── Coil imágenes ─────────────────────────────────────
    implementation(libs.coil.compose)

    // ── Testing ─────────────────────────────────────────
    testImplementation("junit:junit:4.13.2")
    androidTestImplementation("androidx.test.ext:junit:1.2.1")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.6.1")
}
```

---

## 1.7 Variables de entorno — `local.properties`

En el archivo `local.properties`, ubicado en la raíz del proyecto, agrega:

```properties
# Para emulador Android: 10.0.2.2 apunta al localhost de tu máquina
API_BASE_URL=http://10.0.2.2:8000/api/

# Para dispositivo físico, usar la IP local de tu máquina
# API_BASE_URL=http://192.168.1.X:8000/api/
```

> `local.properties` no debe subirse a Git.

En `.gitignore`, verifica que exista:

```gitignore
local.properties
```

---

## 1.8 AndroidManifest.xml — permisos de red

Reemplaza o ajusta `app/src/main/AndroidManifest.xml` así:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <application
        android:name=".ShopApp"
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.Shopapp">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/Theme.Shopapp">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>

        </activity>

    </application>
</manifest>

```

---

## 1.9 Estructura de carpetas objetivo

```text
app/src/main/java/com/shopapp/
├── ShopApplication.kt
├── MainActivity.kt
├── VerificationScreen.kt
│
├── data/
│   ├── remote/
│   │   ├── api/
│   │   ├── dto/
│   │   └── interceptor/
│   ├── local/
│   └── repository/
│
├── domain/
│   ├── model/
│   └── repository/
│
├── presentation/
│   ├── navigation/
│   ├── ui/
│   ├── viewmodel/
│   └── components/
│
├── di/
│
└── theme/
    ├── Color.kt
    ├── Type.kt
    ├── Shape.kt
    └── Theme.kt
```
```javascript
cd app/src/main/java/com/shopapp && \
mkdir -p \
data/{remote/{api,dto,interceptor},local,repository} \
domain/{model,repository} \
presentation/{navigation,ui,viewmodel,components} \
di \
theme && \
touch \
ShopApplication.kt \
MainActivity.kt \
VerificationScreen.kt \
theme/{Color.kt,Type.kt,Shape.kt,Theme.kt}
```

Crear paquetes desde Android Studio:

`com.shopapp` → clic derecho → **New → Package**.

---

## 1.10 Design System — Material 3

### `theme/Color.kt`

```kotlin
package com.shopapp.theme

import androidx.compose.ui.graphics.Color

val Background = Color(0xFF0A0A0F)
val Surface = Color(0xFF111118)
val Surface2 = Color(0xFF1A1A24)
val Border = Color(0xFF2A2A38)
val BorderLight = Color(0xFF1E1E2A)

val TextPrimary = Color(0xFFF0F0F8)
val TextSecondary = Color(0xFF8888AA)
val TextFaint = Color(0xFF44445A)

val Accent = Color(0xFFD4A843)
val AccentLight = Color(0xFFF0C96E)
val AccentDark = Color(0xFFA07820)
val AccentOnDark = Color(0xFF0A0A0F)

val Success = Color(0xFF22C55E)
val Warning = Color(0xFFF59E0B)
val Error = Color(0xFFEF4444)
val Info = Color(0xFF3B82F6)

val StatusPending = Color(0xFFF59E0B)
val StatusConfirmed = Color(0xFF3B82F6)
val StatusShipped = Color(0xFF8B5CF6)
val StatusDelivered = Color(0xFF22C55E)
val StatusCancelled = Color(0xFFEF4444)
```

### `theme/Type.kt`

```kotlin
package com.shopapp.theme

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

val Typography = Typography(
    displayLarge = TextStyle(
        fontWeight = FontWeight.Bold,
        fontSize = 36.sp,
        lineHeight = 44.sp,
        letterSpacing = (-0.5).sp,
    ),
    displayMedium = TextStyle(
        fontWeight = FontWeight.Bold,
        fontSize = 28.sp,
        lineHeight = 36.sp,
    ),
    headlineLarge = TextStyle(
        fontWeight = FontWeight.SemiBold,
        fontSize = 24.sp,
        lineHeight = 32.sp,
    ),
    headlineMedium = TextStyle(
        fontWeight = FontWeight.SemiBold,
        fontSize = 20.sp,
        lineHeight = 28.sp,
    ),
    titleLarge = TextStyle(
        fontWeight = FontWeight.SemiBold,
        fontSize = 18.sp,
        lineHeight = 26.sp,
    ),
    titleMedium = TextStyle(
        fontWeight = FontWeight.Medium,
        fontSize = 16.sp,
        lineHeight = 24.sp,
    ),
    bodyLarge = TextStyle(
        fontWeight = FontWeight.Normal,
        fontSize = 16.sp,
        lineHeight = 24.sp,
    ),
    bodyMedium = TextStyle(
        fontWeight = FontWeight.Normal,
        fontSize = 14.sp,
        lineHeight = 20.sp,
    ),
    bodySmall = TextStyle(
        fontWeight = FontWeight.Normal,
        fontSize = 12.sp,
        lineHeight = 16.sp,
    ),
    labelLarge = TextStyle(
        fontWeight = FontWeight.SemiBold,
        fontSize = 14.sp,
        letterSpacing = 0.5.sp,
    ),
    labelSmall = TextStyle(
        fontWeight = FontWeight.Bold,
        fontSize = 11.sp,
        letterSpacing = 0.8.sp,
    ),
)
```

### `theme/Shape.kt`

```kotlin
package com.shopapp.theme

import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Shapes
import androidx.compose.ui.unit.dp

val Shapes = Shapes(
    extraSmall = RoundedCornerShape(6.dp),
    small = RoundedCornerShape(10.dp),
    medium = RoundedCornerShape(14.dp),
    large = RoundedCornerShape(18.dp),
    extraLarge = RoundedCornerShape(24.dp),
)
```

### `theme/Theme.kt`

```kotlin
package com.shopapp.theme

import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.darkColorScheme
import androidx.compose.runtime.Composable
import androidx.compose.ui.graphics.Color

private val DarkColorScheme = darkColorScheme(
    primary = Accent,
    onPrimary = AccentOnDark,
    primaryContainer = Surface2,
    secondary = AccentLight,
    onSecondary = AccentOnDark,
    background = Background,
    onBackground = TextPrimary,
    surface = Surface,
    onSurface = TextPrimary,
    surfaceVariant = Surface2,
    onSurfaceVariant = TextSecondary,
    outline = Border,
    outlineVariant = BorderLight,
    error = Error,
    onError = Color.White,
)

@Composable
fun ShopAppTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = DarkColorScheme,
        typography = Typography,
        shapes = Shapes,
        content = content,
    )
}
```

---

## 1.11 Modelos de dominio — `domain/model/`

### `domain/model/Auth.kt`

```kotlin
package com.shopapp.domain.model

data class AuthTokens(
    val access: String,
    val refresh: String,
)

data class LoggedUser(
    val id: Int,
    val username: String,
    val email: String,
    val isStaff: Boolean,
)
```

### `domain/model/Category.kt`

```kotlin
package com.shopapp.domain.model

data class Category(
    val id: Int,
    val name: String,
    val slug: String,
    val description: String,
    val isActive: Boolean,
    val totalProducts: Int,
    val createdAt: String,
)

data class CategoryPayload(
    val name: String,
    val slug: String,
    val description: String,
    val isActive: Boolean,
)
```

### `domain/model/Product.kt`

```kotlin
package com.shopapp.domain.model

data class Product(
    val id: Int,
    val name: String,
    val description: String,
    val price: Double,
    val priceWithTax: Double,
    val stock: Int,
    val inStock: Boolean,
    val isActive: Boolean,
    val imageUrl: String?,
    val categoryId: Int?,
    val categoryName: String?,
    val createdAt: String,
    val updatedAt: String,
)

data class ProductPayload(
    val name: String,
    val description: String,
    val price: Double,
    val stock: Int,
    val isActive: Boolean,
    val categoryId: Int,
)

data class ProductFilters(
    val search: String? = null,
    val category: Int? = null,
    val priceMin: Double? = null,
    val priceMax: Double? = null,
    val stockMin: Int? = null,
    val isActive: Boolean? = null,
    val ordering: String? = null,
    val page: Int = 1,
    val pageSize: Int = 12,
)
```

### `domain/model/Order.kt`

```kotlin
package com.shopapp.domain.model

enum class OrderStatus(val value: String, val label: String) {
    PENDING("pending", "Pendiente"),
    CONFIRMED("confirmed", "Confirmado"),
    SHIPPED("shipped", "Enviado"),
    DELIVERED("delivered", "Entregado"),
    CANCELLED("cancelled", "Cancelado");

    companion object {
        fun fromValue(value: String): OrderStatus =
            entries.firstOrNull { it.value == value } ?: PENDING
    }
}

data class OrderItem(
    val id: Int,
    val productId: Int,
    val productName: String,
    val quantity: Int,
    val unitPrice: Double,
    val subtotal: Double,
)

data class Order(
    val id: Int,
    val username: String,
    val status: OrderStatus,
    val total: Double,
    val numItems: Int,
    val items: List<OrderItem>,
    val createdAt: String,
    val updatedAt: String,
)
```

### `domain/model/User.kt`

```kotlin
package com.shopapp.domain.model

data class User(
    val id: Int,
    val username: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val isStaff: Boolean,
    val isActive: Boolean,
    val dateJoined: String,
    val numOrders: Int,
)

data class UserPayload(
    val username: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val isStaff: Boolean,
    val isActive: Boolean,
    val password: String? = null,
)
```

---

## 1.12 Clase Application con Hilt

Crea el archivo `ShopApplication.kt` en `app/src/main/java/com/shopapp/`:

```kotlin
package com.shopapp

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class ShopApp : Application()
```

---

## 1.13 MainActivity — punto de entrada

Reemplaza `MainActivity.kt` por este contenido:

```kotlin
package com.shopapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Surface
import androidx.compose.ui.Modifier
import com.shopapp.theme.ShopAppTheme
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            ShopAppTheme {
                Surface(modifier = Modifier.fillMaxSize()) {
                    VerificationScreen()
                }
            }
        }
    }
}
```

---

## 1.14 Pantalla de verificación — `VerificationScreen.kt`

Crea el archivo `VerificationScreen.kt` en `app/src/main/java/com/shopapp/`:

```kotlin
package com.shopapp

import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.HorizontalDivider
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.tooling.preview.Preview
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.shopapp.theme.Accent
import com.shopapp.theme.Background
import com.shopapp.theme.Border
import com.shopapp.theme.BorderLight
import com.shopapp.theme.Error
import com.shopapp.theme.Info
import com.shopapp.theme.ShopAppTheme
import com.shopapp.theme.Success
import com.shopapp.theme.Surface
import com.shopapp.theme.TextFaint
import com.shopapp.theme.TextPrimary
import com.shopapp.theme.TextSecondary
import com.shopapp.theme.Warning

@Composable
fun VerificationScreen() {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(Background),
    ) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .verticalScroll(rememberScrollState())
                .padding(24.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
            verticalArrangement = Arrangement.Center,
        ) {
            Text(
                text = "ShopApp",
                fontSize = 40.sp,
                fontWeight = FontWeight.Bold,
                color = Accent,
                modifier = Modifier.padding(bottom = 8.dp),
            )

            Text(
                text = "Módulo 1 · Android + Jetpack Compose",
                style = MaterialTheme.typography.bodyMedium,
                color = TextSecondary,
                modifier = Modifier.padding(bottom = 32.dp),
            )

            EnvCard(
                items = listOf(
                    "Kotlin" to "2.0.21",
                    "Compose BOM" to "2024.10.01",
                    "Material 3" to "✓",
                    "Hilt" to "2.52",
                    "Retrofit" to "2.11.0",
                    "API URL" to BuildConfig.API_BASE_URL,
                ),
            )

            Spacer(modifier = Modifier.height(24.dp))

            Text(
                text = "Design System",
                style = MaterialTheme.typography.labelSmall,
                color = TextSecondary,
                letterSpacing = 1.sp,
                modifier = Modifier
                    .padding(bottom = 12.dp)
                    .align(Alignment.Start),
            )

            Row(
                horizontalArrangement = Arrangement.spacedBy(8.dp),
                modifier = Modifier.fillMaxWidth(),
            ) {
                listOf(
                    "Accent" to Accent,
                    "Success" to Success,
                    "Warning" to Warning,
                    "Error" to Error,
                    "Info" to Info,
                ).forEach { (label, color) ->
                    Column(
                        horizontalAlignment = Alignment.CenterHorizontally,
                        modifier = Modifier.weight(1f),
                    ) {
                        Box(
                            modifier = Modifier
                                .size(44.dp)
                                .background(color, RoundedCornerShape(8.dp)),
                        )
                        Text(
                            text = label,
                            fontSize = 9.sp,
                            color = TextFaint,
                            modifier = Modifier.padding(top = 4.dp),
                        )
                    }
                }
            }

            Spacer(modifier = Modifier.height(24.dp))

            Text(
                text = "Modelos de dominio",
                style = MaterialTheme.typography.labelSmall,
                color = TextSecondary,
                letterSpacing = 1.sp,
                modifier = Modifier
                    .padding(bottom = 12.dp)
                    .align(Alignment.Start),
            )

            listOf("Auth.kt", "Category.kt", "Product.kt", "Order.kt", "User.kt").forEach { file ->
                Row(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(vertical = 4.dp),
                    horizontalArrangement = Arrangement.SpaceBetween,
                ) {
                    Text(
                        text = "domain/model/$file",
                        style = MaterialTheme.typography.bodySmall,
                        color = Accent,
                        fontWeight = FontWeight.Medium,
                    )
                    Text(
                        text = "✓",
                        color = Success,
                        style = MaterialTheme.typography.bodySmall,
                    )
                }
            }
        }
    }
}

@Composable
private fun EnvCard(items: List<Pair<String, String>>) {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .background(Surface, RoundedCornerShape(16.dp))
            .border(1.dp, Border, RoundedCornerShape(16.dp))
            .padding(16.dp),
    ) {
        Text(
            text = "Estado del entorno",
            style = MaterialTheme.typography.labelSmall,
            color = TextSecondary,
            letterSpacing = 1.sp,
            modifier = Modifier.padding(bottom = 12.dp),
        )

        items.forEachIndexed { index, item ->
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(vertical = 8.dp),
                horizontalArrangement = Arrangement.SpaceBetween,
            ) {
                Text(
                    text = item.first,
                    style = MaterialTheme.typography.bodySmall,
                    color = TextSecondary,
                )
                Text(
                    text = item.second,
                    style = MaterialTheme.typography.bodySmall,
                    color = TextPrimary,
                    fontWeight = FontWeight.SemiBold,
                    maxLines = 1,
                )
            }

            if (index < items.lastIndex) {
                HorizontalDivider(color = BorderLight, thickness = 0.5.dp)
            }
        }
    }
}

@Preview(showBackground = true)
@Composable
fun VerificationScreenPreview() {
    ShopAppTheme {
        VerificationScreen()
    }
}
```

---

## 1.15 Limpiar y sincronizar Gradle

Después de reemplazar los archivos de Gradle, ejecuta:

```bash
./gradlew --stop
./gradlew clean
./gradlew build
```

En Android Studio también puedes hacer:

```text
File → Invalidate Caches / Restart
Sync Project with Gradle Files
```

---

## ✅ Checkpoint Módulo 1

### Compilar y ejecutar

```bash
./gradlew assembleDebug
```

O ejecutar directamente desde Android Studio con el botón **Run**.

| Verificación            | Resultado esperado                      |
| ----------------------- | --------------------------------------- |
| Compila sin errores     | 0 errores en Build output               |
| App arranca en emulador | Pantalla oscura con "ShopApp" en dorado |
| API URL visible         | `http://10.0.2.2:8000/api/`             |
| Paleta de colores       | 5 chips coloreados                      |
| Modelos de dominio      | 5 archivos con ✓ verde                  |

---

## Problemas comunes

### Error: plugin already on the classpath

```text
The request for this plugin could not be satisfied because the plugin is already on the classpath with an unknown version
```

Solución:

* Eliminar cualquier bloque `buildscript { ... }`.
* No usar `classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:...")`.
* Usar solamente `plugins { alias(libs.plugins...) }`.

---

### Error: Cannot resolve external dependency kotlin-gradle-plugin

```text
Cannot resolve external dependency org.jetbrains.kotlin:kotlin-gradle-plugin:... because no repositories are defined
```

Solución:

* Eliminar el `classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:...")`.
* No usar `...` como versión.
* Mantener los repositorios en `settings.gradle.kts`.

---

### Error: Hilt requires @HiltAndroidApp on Application

Solución:

* Verificar que existe `ShopApplication.kt`.
* Verificar que tiene `@HiltAndroidApp`.
* Verificar que `AndroidManifest.xml` tiene:

```xml
android:name=".ShopApplication"
```

---

### Error: Unresolved reference BuildConfig

Solución:

* Verificar en `app/build.gradle.kts`:

```kotlin
buildFeatures {
    buildConfig = true
}
```

Luego ejecutar:

```bash
./gradlew clean
./gradlew build
```

---

## Resumen

| Elemento                                            | Estado |
| --------------------------------------------------- | ------ |
| Proyecto Android con Compose + Material 3           | ✅      |
| Gradle configurado sin `buildscript` antiguo        | ✅      |
| Version Catalog configurado                         | ✅      |
| Hilt configurado para DI                            | ✅      |
| Retrofit + OkHttp en dependencias                   | ✅      |
| DataStore en dependencias                           | ✅      |
| Coil para imágenes                                  | ✅      |
| Design system dark + accent dorado                  | ✅      |
| Tipografía Material 3 personalizada                 | ✅      |
| `domain/model/Auth.kt`                              | ✅      |
| `domain/model/Category.kt`                          | ✅      |
| `domain/model/Product.kt`                           | ✅      |
| `domain/model/Order.kt` con enum                    | ✅      |
| `domain/model/User.kt`                              | ✅      |
| `BuildConfig.API_BASE_URL` desde `local.properties` | ✅      |

**Siguiente módulo →** M2: Retrofit + interceptores JWT + DTOs + repositorios + DataStore
