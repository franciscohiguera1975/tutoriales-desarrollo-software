# ShopApp Android — Módulo 12
## Imágenes: foto de producto y avatar de usuario

> **Objetivo:** Subir imágenes desde la galería del dispositivo y mostrarlas en la app usando Coil para la carga asíncrona, `ActivityResultContracts.GetContent` para el selector de archivos y Retrofit multipart para el envío al backend.

| Función | ¿Quién? | Pantalla |
|---------|---------|---------|
| Ver foto del producto | Todos (público) | `ProductDetailScreen` ← ya implementado |
| Subir / cambiar foto del producto | Solo staff | `ProductFormSheet` (ícono lápiz) |
| Ver y cambiar avatar | Usuario autenticado | `ProfileScreen` |

---

## 12.1 Dependencias

### `gradle/libs.versions.toml`

```toml
# gradle/libs.versions.toml

[versions]
# ... versiones existentes ...
coil = "2.7.0"

[libraries]
# ... librerías existentes ...
coil-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }
```

### `app/build.gradle.kts`

`app/build.gradle.kts`
```kotlin
dependencies {
    // ... dependencias existentes ...
    implementation(libs.coil.compose)
}
```

Sincroniza el proyecto con **File → Sync Project with Gradle Files** antes de continuar.

---

## ── Parte 1 · Foto de producto ──────────────────────────────────────────────

## 12.2 DTOs y modelos — imageUrl ya está listo ✓

El campo `imageUrl` ya existe en `ProductDto` y en el modelo de dominio `Product`. No se requieren cambios.

`data/remote/dto/ProductDto.kt — fragmento (ya existe, verificar)`
```kotlin
@SerializedName("image_url") val imageUrl: String?,
```

`domain/model/Product.kt — fragmento (ya existe, verificar)`
```kotlin
val imageUrl: String?,   // null si el producto no tiene imagen aún
```

> El mapper `ProductDto.toDomain()` ya incluye `imageUrl = imageUrl`. Solo confirma que está presente antes de continuar.

---

## 12.3 Endpoint multipart en `ProductApi`

Añade al final de la interfaz existente:

`data/remote/api/ProductApi.kt`
```kotlin
package com.shopapp.data.remote.api

// ... imports existentes ...
import okhttp3.MultipartBody
import retrofit2.Response

interface ProductApi {

    // ── Endpoints existentes ────────────────────────────────────────────────

    /**
     * Sube o reemplaza la imagen de un producto.
     * Solo accesible para usuarios con is_staff = true.
     * Backend: PATCH /api/products/{id}/  multipart/form-data campo "image"
     */
    @Multipart
    @PATCH("products/{id}/")
    suspend fun uploadProductImage(
        @Path("id") id: Int,
        @Part image: MultipartBody.Part,
    ): Response<ProductDto>
}
```

---

## 12.4 `ProductRepository` — añadir `uploadProductImage`

### Interfaz — `domain/repository/ProductRepository.kt`

`domain/repository/ProductRepository.kt`
```kotlin
package com.shopapp.domain.repository

import android.net.Uri
import com.shopapp.domain.model.Product

interface ProductRepository {
    // ... métodos existentes ...

    /** Sube una imagen para el producto indicado. Devuelve la URL absoluta resultante. */
    suspend fun uploadProductImage(id: Int, uri: Uri): Result<String>
}
```

### Implementación — `data/repository/ProductRepositoryImpl.kt`

`data/repository/ProductRepositoryImpl.kt`
```kotlin
package com.shopapp.data.repository

import android.content.Context
import android.net.Uri
import com.shopapp.data.remote.api.ProductApi
import com.shopapp.domain.repository.ProductRepository
import dagger.hilt.android.qualifiers.ApplicationContext
import okhttp3.MediaType.Companion.toMediaTypeOrNull
import okhttp3.MultipartBody
import okhttp3.RequestBody.Companion.toRequestBody
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class ProductRepositoryImpl @Inject constructor(
    private val api: ProductApi,
    @ApplicationContext private val context: Context,
) : ProductRepository {

    // ... implementaciones existentes ...

    override suspend fun uploadProductImage(id: Int, uri: Uri): Result<String> =
        runCatching {
            val part     = uri.toMultipart(context, fieldName = "image")
            val response = api.uploadProductImage(id, part)
            if (response.isSuccessful) {
                response.body()?.imageUrl ?: error("El servidor no devolvió una URL de imagen")
            } else {
                error(response.errorBody()?.string() ?: "Error ${response.code()}")
            }
        }
}

// ── Extensión interna reutilizada por ambos repositorios ──────────────────────

/**
 * Convierte una URI de la galería en un [MultipartBody.Part] listo para Retrofit.
 * Detecta el MIME type real del archivo; si no puede determinarlo usa image/jpeg.
 */
internal fun Uri.toMultipart(context: Context, fieldName: String): MultipartBody.Part {
    val resolver    = context.contentResolver
    val mimeType    = resolver.getType(this) ?: "image/jpeg"
    val bytes       = resolver.openInputStream(this)?.readBytes()
        ?: error("No se pudo leer el archivo seleccionado")
    val requestBody = bytes.toRequestBody(mimeType.toMediaTypeOrNull())
    val fileName    = "upload.${mimeType.substringAfterLast('/')}"
    return MultipartBody.Part.createFormData(fieldName, fileName, requestBody)
}
```

---

## 12.5 `ProductImageViewModel` — ya existe ✓

> **Este ViewModel ya existe en el proyecto.** Se muestra solo como referencia.

`presentation/viewmodel/ProductImageViewModel.kt  — ya existe, no crear`
```kotlin
package com.shopapp.presentation.viewmodel

import android.net.Uri
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.repository.ProductRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class ProductImageViewModel @Inject constructor(
    private val repository: ProductRepository,
) : ViewModel() {

    private val _isUploading = MutableStateFlow(false)
    val isUploading: StateFlow<Boolean> = _isUploading.asStateFlow()

    private val _uploadResult = MutableStateFlow<Result<String>?>(null)
    val uploadResult: StateFlow<Result<String>?> = _uploadResult.asStateFlow()

    fun uploadImage(productId: Int, uri: Uri) {
        if (_isUploading.value) return
        viewModelScope.launch {
            _isUploading.value  = true
            _uploadResult.value = null
            _uploadResult.value = repository.uploadProductImage(productId, uri)
            _isUploading.value  = false
        }
    }

    fun clearResult() { _uploadResult.value = null }
}
```

---

## 12.6 Composable `ProductImageSection` (admin)

### `presentation/ui/admin/products/ProductImageSection.kt`

Muestra la imagen del producto con Coil. Si `isStaff = true`, superpone un botón de cámara para reemplazar la imagen. Gestiona el selector de archivos y delega la subida al `ProductImageViewModel`.

`presentation/ui/admin/products/ProductImageSection.kt`
```kotlin
package com.shopapp.presentation.ui.admin.products

import android.net.Uri
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.animation.AnimatedVisibility
import androidx.compose.animation.fadeIn
import androidx.compose.animation.fadeOut
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.CameraAlt
import androidx.compose.material.icons.filled.BrokenImage
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import coil.compose.SubcomposeAsyncImage
import coil.request.ImageRequest
import com.shopapp.presentation.viewmodel.ProductImageViewModel

@Composable
fun ProductImageSection(
    productId:       Int,
    currentImageUrl: String?,          // URL actual del producto
    isStaff:         Boolean,
    onImageUpdated:  () -> Unit,       // llamado tras subida exitosa
    modifier:        Modifier = Modifier,
    imageViewModel:  ProductImageViewModel = hiltViewModel(),
) {
    val isUploading  by imageViewModel.isUploading.collectAsState()
    val uploadResult by imageViewModel.uploadResult.collectAsState()

    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(uploadResult) {
        uploadResult?.let { result ->
            result
                .onSuccess {
                    onImageUpdated()
                    snackbarHostState.showSnackbar("Imagen actualizada correctamente")
                }
                .onFailure { e ->
                    snackbarHostState.showSnackbar(
                        "Error al subir imagen: ${e.message ?: "desconocido"}"
                    )
                }
            imageViewModel.clearResult()
        }
    }

    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.GetContent()
    ) { uri: Uri? ->
        uri?.let { imageViewModel.uploadImage(productId, it) }
    }

    Box(modifier = modifier) {
        // ── Imagen principal ──────────────────────────────────────────────────
        if (!currentImageUrl.isNullOrBlank()) {
            SubcomposeAsyncImage(
                model = ImageRequest.Builder(LocalContext.current)
                    .data(currentImageUrl)
                    .crossfade(true)
                    .build(),
                contentDescription = "Imagen del producto",
                contentScale       = ContentScale.Crop,
                modifier           = Modifier.fillMaxSize(),
                loading = {
                    Box(
                        contentAlignment = Alignment.Center,
                        modifier         = Modifier.fillMaxSize(),
                    ) { CircularProgressIndicator() }
                },
                error = {
                    Box(
                        contentAlignment = Alignment.Center,
                        modifier         = Modifier.fillMaxSize().background(Color.DarkGray),
                    ) {
                        Icon(
                            imageVector        = Icons.Default.BrokenImage,
                            contentDescription = null,
                            tint               = Color.White,
                            modifier           = Modifier.size(48.dp),
                        )
                    }
                },
            )
        } else {
            Box(
                contentAlignment = Alignment.Center,
                modifier         = Modifier.fillMaxSize().background(Color.DarkGray),
            ) {
                Icon(
                    imageVector        = Icons.Default.BrokenImage,
                    contentDescription = null,
                    tint               = Color.White,
                    modifier           = Modifier.size(48.dp),
                )
            }
        }

        // ── Overlay de carga ──────────────────────────────────────────────────
        AnimatedVisibility(
            visible  = isUploading,
            enter    = fadeIn(),
            exit     = fadeOut(),
            modifier = Modifier.fillMaxSize(),
        ) {
            Box(
                contentAlignment = Alignment.Center,
                modifier         = Modifier
                    .fillMaxSize()
                    .background(Color.Black.copy(alpha = 0.55f)),
            ) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    CircularProgressIndicator(color = Color.White)
                    Spacer(Modifier.height(8.dp))
                    Text("Subiendo imagen...", color = Color.White)
                }
            }
        }

        // ── Botón cámara (solo staff) ─────────────────────────────────────────
        if (isStaff && !isUploading) {
            FloatingActionButton(
                onClick   = { launcher.launch("image/*") },
                modifier  = Modifier
                    .align(Alignment.BottomEnd)
                    .padding(12.dp)
                    .size(40.dp),
                shape          = CircleShape,
                containerColor = MaterialTheme.colorScheme.primary,
            ) {
                Icon(
                    imageVector        = Icons.Default.CameraAlt,
                    contentDescription = "Cambiar imagen",
                    tint               = Color.White,
                    modifier           = Modifier.size(20.dp),
                )
            }
        }

        SnackbarHost(
            hostState = snackbarHostState,
            modifier  = Modifier.align(Alignment.BottomCenter),
        )
    }
}
```

---

## 12.7 Integrar `ProductImageSection` en `ProductFormSheet` (admin)

La foto se gestiona desde el **bottom sheet de edición** — el que se abre al pulsar el ícono de lápiz en `ProductsAdminScreen`. No hace falta una pantalla nueva.

### Paso 1 — Añadir parámetro `onImageUpdated` a `ProductFormSheet`

`presentation/ui/admin/products/ProductFormSheet.kt  — cambios en la firma`
```kotlin

@Composable
fun ProductFormSheet(
    initial:        Product?,
    categories:     List<Category>,
    formState:      ProductFormState,
    onSave:         (ProductPayload) -> Unit,
    onDismiss:      () -> Unit,
    onImageUpdated: () -> Unit = {},   // ← nuevo parámetro
) {
    val isEdit = initial != null
    // ... resto del código sin cambios ...
```

### Paso 2 — Añadir `ProductImageSection` al inicio del sheet

Dentro del `Column` que ya existe, añade la sección de imagen **antes** de los campos del formulario (solo al editar):

`presentation/ui/admin/products/ProductFormSheet.kt — dentro del Column del sheet`
```kotlin

// ── Imagen del producto (solo al editar) ─────────────────────────────────────
if (isEdit && initial != null) {
    ProductImageSection(
        productId       = initial.id,
        currentImageUrl = initial.imageUrl,
        isStaff         = true,         // solo staff llega hasta aquí
        onImageUpdated  = onImageUpdated,
        modifier        = Modifier
            .fillMaxWidth()
            .height(220.dp),
    )
    Spacer(Modifier.height(8.dp))
}

// ── Nombre ───────────────────────────────────────────────────────────────────
// ... campos existentes continúan aquí ...
```

### Paso 3 — Pasar el callback desde `ProductsAdminScreen`

`presentation/ui/admin/products/ProductsAdminScreen.kt — fragmento relevante`
```kotlin

ProductFormSheet(
    initial        = editTarget,
    categories     = categories,
    formState      = formState,
    onSave         = { payload -> viewModel.save(editTarget?.id, payload) },
    onDismiss      = { showForm = false; editTarget = null },
    onImageUpdated = { viewModel.load() },   // ← nuevo
)
```

> **Flujo completo:** staff pulsa el lápiz → sheet se abre → en la parte superior aparece la imagen actual con botón de cámara → al seleccionar una foto se sube automáticamente → Snackbar confirma el resultado → `viewModel.load()` recarga la lista con la nueva URL.

---

## 12.8 `ProductDetailScreen` — visualización pública ✓

> **No requiere ningún cambio.** La pantalla pública ya muestra la imagen del producto usando Coil.

`ProductDetailScreen` (en `ui/uipublic/product/`) ya implementa:

`presentation/ui/uipublic/product/ProductDetailScreen.kt — fragmento (ya existe)`
```kotlin

Box(modifier = Modifier.fillMaxWidth().height(320.dp)) {
    if (product.imageUrl != null) {
        AsyncImage(
            model              = product.imageUrl,
            contentDescription = product.name,
            contentScale       = ContentScale.Crop,
            modifier           = Modifier.fillMaxSize(),
        )
    } else {
        // Placeholder con emoji 📦
        Box(...) { Text("📦", fontSize = 72.sp) }
    }
}
```

Cuando el staff suba una foto desde `ProductFormSheet`, la URL se almacena en el backend. La próxima vez que un cliente abra `ProductDetailScreen`, Coil la descargará y la mostrará automáticamente.

---

## 🚦 PARADA 1 — Foto de producto

> **Qué está listo:** endpoint multipart en `ProductApi`, `uploadProductImage` en `ProductRepository`, `ProductImageSection`, integración en `ProductFormSheet`, visualización en `ProductDetailScreen`.
> **Qué vas a probar:**

**Prueba 1 — Subir imagen como staff**

1. Autenticarse como usuario **staff**.
2. Ir a la sección **Productos** en el panel de administración.
3. Pulsar el ícono de lápiz ✏️ en cualquier producto → se abre el sheet de edición.
4. En la parte superior del sheet aparece la imagen actual (o el ícono gris si no tiene imagen).
5. Pulsar el botón de cámara 📷 (esquina inferior derecha de la imagen).
6. Seleccionar una imagen JPEG o PNG de la galería (menor de 2 MB).
7. Resultado esperado:
   - Overlay "Subiendo imagen..." con `CircularProgressIndicator`.
   - Snackbar: **"Imagen actualizada correctamente"**.
   - La lista de productos se recarga y el thumbnail del producto muestra la nueva imagen.

**Prueba 2 — Ver imagen como cliente**

1. Cerrar sesión. Entrar como cliente.
2. Abrir el catálogo y pulsar el producto al que se subió la imagen.
3. En `ProductDetailScreen`, la imagen debe mostrarse en el banner superior de 320 dp.

**Casos de borde:**

| Acción | Resultado esperado |
|--------|--------------------|
| Imagen mayor de 2 MB | Error `400` del backend mostrado en Snackbar |
| Formato no soportado (PDF, GIF) | Error `400` del backend mostrado en Snackbar |
| Sin conexión | Snackbar con el mensaje de error de red |

---

## ── Parte 2 · Avatar de usuario ────────────────────────────────────────────

## 12.9 DTOs y modelos — añadir `avatarUrl`

`UserDto` y el modelo `User` no tienen el campo `avatarUrl` todavía. Añádelo:

`data/remote/dto/UserDto.kt — añadir campo avatar_url`
```kotlin
import com.google.gson.annotations.SerializedName

data class UserDto(
    val id:         Int,
    val username:   String,
    val email:      String,
    @SerializedName("first_name")  val firstName:  String,
    @SerializedName("last_name")   val lastName:   String,
    @SerializedName("is_staff")    val isStaff:    Boolean,
    @SerializedName("is_active")   val isActive:   Boolean,
    @SerializedName("date_joined") val dateJoined: String,
    @SerializedName("num_orders")  val numOrders:  Int,
    @SerializedName("avatar_url")
    val avatarUrl:  String? = null,    // ← nuevo campo
)
```

`domain/model/User.kt — añadir avatarUrl`
```kotlin
data class User(
    val id:         Int,
    val username:   String,
    val email:      String,
    val firstName:  String,
    val lastName:   String,
    val isStaff:    Boolean,
    val isActive:   Boolean,
    val dateJoined: String,
    val numOrders:  Int,
    val avatarUrl:  String? = null,    // ← nuevo campo
)
```

`data/remote/dto/UserDto.kt — actualizar toDomain()`
```kotlin
fun UserDto.toDomain() = User(
    id          = id,
    username    = username,
    email       = email,
    firstName   = firstName,
    lastName    = lastName,
    isStaff     = isStaff,
    isActive    = isActive,
    dateJoined  = dateJoined,
    numOrders   = numOrders,
    avatarUrl   = avatarUrl,           // ← nuevo campo
)
```

---

## 12.10 Endpoints en `UserApi`

Añade al final de la interfaz existente:

`data/remote/api/UserApi.kt`
```kotlin
package com.shopapp.data.remote.api

// ... imports existentes ...
import okhttp3.MultipartBody
import retrofit2.Response

interface UserApi {

    // ── Endpoints existentes ────────────────────────────────────────────────

    /** Obtiene el perfil del usuario autenticado. Backend: GET /api/users/profile/ */
    @GET("users/profile/")
    suspend fun getProfile(): Response<UserDto>

    /**
     * Sube o reemplaza el avatar del usuario autenticado.
     * Backend: PATCH /api/users/profile/  multipart/form-data campo "avatar"
     */
    @Multipart
    @PATCH("users/profile/")
    suspend fun uploadAvatar(
        @Part avatar: MultipartBody.Part,
    ): Response<UserDto>
}
```

---

## 12.11 `UserRepository` — añadir `getProfile` y `uploadAvatar`

### Interfaz — `domain/repository/UserRepository.kt`

`domain/repository/UserRepository.kt`
```kotlin
package com.shopapp.domain.repository

import android.net.Uri
import com.shopapp.domain.model.User

interface UserRepository {
    // ... métodos existentes ...

    /** Obtiene el perfil del usuario autenticado. */
    suspend fun getProfile(): Result<User>

    /** Sube o reemplaza el avatar. Devuelve la URL absoluta resultante. */
    suspend fun uploadAvatar(uri: Uri): Result<String>
}
```

### Implementación — `data/repository/UserRepositoryImpl.kt`

`data/repository/UserRepositoryImpl.kt`
```kotlin
package com.shopapp.data.repository

import android.content.Context
import android.net.Uri
import com.shopapp.data.remote.api.UserApi
import com.shopapp.data.remote.dto.toDomain
import com.shopapp.domain.model.User
import com.shopapp.domain.repository.UserRepository
import dagger.hilt.android.qualifiers.ApplicationContext
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class UserRepositoryImpl @Inject constructor(
    private val api: UserApi,
    @ApplicationContext private val context: Context,
) : UserRepository {

    // ... implementaciones existentes ...

    override suspend fun getProfile(): Result<User> = runCatching {
        val response = api.getProfile()
        if (response.isSuccessful) response.body()!!.toDomain()
        else error(response.errorBody()?.string() ?: "Error ${response.code()}")
    }

    override suspend fun uploadAvatar(uri: Uri): Result<String> = runCatching {
        val part     = uri.toMultipart(context, fieldName = "avatar")
        val response = api.uploadAvatar(part)
        if (response.isSuccessful) {
            response.body()?.avatarUrl ?: error("El servidor no devolvió una URL de avatar")
        } else {
            error(response.errorBody()?.string() ?: "Error ${response.code()}")
        }
    }
}
```

> La función `Uri.toMultipart` declarada en `ProductRepositoryImpl.kt` tiene visibilidad `internal` y es accesible desde cualquier archivo del módulo `data`.

---

## 12.12 `ProfileViewModel` — crear

### `presentation/viewmodel/ProfileViewModel.kt`

`presentation/viewmodel/ProfileViewModel.kt`
```kotlin
package com.shopapp.presentation.viewmodel

import android.net.Uri
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.User
import com.shopapp.domain.repository.UserRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

data class ProfileUiState(
    val profile:     User?    = null,
    val isLoading:   Boolean  = false,
    val isUploading: Boolean  = false,
    val error:       String?  = null,
    val avatarUrl:   String?  = null,
)

@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val repository: UserRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(ProfileUiState())
    val state: StateFlow<ProfileUiState> = _state.asStateFlow()

    private val _snackbar = MutableSharedFlow<String>(extraBufferCapacity = 1)
    val snackbar: SharedFlow<String> = _snackbar.asSharedFlow()

    init { loadProfile() }

    fun loadProfile() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            repository.getProfile()
                .onSuccess { profile ->
                    _state.update {
                        it.copy(
                            profile   = profile,
                            avatarUrl = profile.avatarUrl,
                            isLoading = false,
                        )
                    }
                }
                .onFailure { e ->
                    _state.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    fun uploadAvatar(uri: Uri) {
        if (_state.value.isUploading) return
        viewModelScope.launch {
            _state.update { it.copy(isUploading = true) }
            repository.uploadAvatar(uri)
                .onSuccess { url ->
                    _state.update { it.copy(isUploading = false, avatarUrl = url) }
                    _snackbar.tryEmit("Avatar actualizado correctamente")
                }
                .onFailure { e ->
                    _state.update { it.copy(isUploading = false) }
                    _snackbar.tryEmit("Error al subir el avatar: ${e.message ?: "desconocido"}")
                }
        }
    }
}
```

---

## 12.13 `AvatarSection` — composable

### `presentation/ui/client/profile/AvatarSection.kt`

Muestra el avatar circular del usuario. Si no hay avatar, renderiza las iniciales sobre un fondo derivado del nombre de usuario. Al pulsarlo se abre el selector de imágenes.

`presentation/ui/client/profile/AvatarSection.kt`
```kotlin
package com.shopapp.presentation.ui.client.profile

import android.net.Uri
import androidx.activity.compose.rememberLauncherForActivityResult
import androidx.activity.result.contract.ActivityResultContracts
import androidx.compose.animation.AnimatedVisibility
import androidx.compose.animation.fadeIn
import androidx.compose.animation.fadeOut
import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.CameraAlt
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import coil.compose.SubcomposeAsyncImage
import coil.request.ImageRequest

/**
 * Avatar circular del usuario con selector de imagen al pulsarlo.
 *
 * @param avatarUrl       URL absoluta del avatar actual (puede ser null).
 * @param username        Nombre de usuario para calcular iniciales y color de fondo.
 * @param isUploading     true mientras se está enviando el nuevo avatar al servidor.
 * @param onImageSelected Se invoca con la URI seleccionada por el usuario.
 */
@Composable
fun AvatarSection(
    avatarUrl:       String?,
    username:        String,
    isUploading:     Boolean,
    onImageSelected: (Uri) -> Unit,
    modifier:        Modifier = Modifier,
) {
    val launcher = rememberLauncherForActivityResult(
        contract = ActivityResultContracts.GetContent()
    ) { uri: Uri? ->
        uri?.let { onImageSelected(it) }
    }

    Box(
        contentAlignment = Alignment.BottomEnd,
        modifier         = modifier.size(100.dp),
    ) {
        Box(
            modifier = Modifier
                .size(100.dp)
                .clip(CircleShape)
                .border(2.dp, MaterialTheme.colorScheme.primary, CircleShape)
                .clickable(enabled = !isUploading) { launcher.launch("image/*") },
        ) {
            if (!avatarUrl.isNullOrBlank()) {
                SubcomposeAsyncImage(
                    model = ImageRequest.Builder(LocalContext.current)
                        .data(avatarUrl)
                        .crossfade(true)
                        .build(),
                    contentDescription = "Avatar de $username",
                    contentScale       = ContentScale.Crop,
                    modifier           = Modifier.fillMaxSize(),
                    loading = {
                        Box(
                            contentAlignment = Alignment.Center,
                            modifier         = Modifier
                                .fillMaxSize()
                                .background(avatarBackground(username)),
                        ) {
                            CircularProgressIndicator(
                                modifier    = Modifier.size(24.dp),
                                color       = Color.White,
                                strokeWidth = 2.dp,
                            )
                        }
                    },
                    error = { InitialsAvatar(username) },
                )
            } else {
                InitialsAvatar(username)
            }

            AnimatedVisibility(
                visible  = isUploading,
                enter    = fadeIn(),
                exit     = fadeOut(),
                modifier = Modifier.fillMaxSize(),
            ) {
                Box(
                    contentAlignment = Alignment.Center,
                    modifier         = Modifier
                        .fillMaxSize()
                        .background(Color.Black.copy(alpha = 0.50f)),
                ) {
                    CircularProgressIndicator(
                        color       = Color.White,
                        modifier    = Modifier.size(28.dp),
                        strokeWidth = 2.dp,
                    )
                }
            }
        }

        if (!isUploading) {
            Surface(
                shape          = CircleShape,
                color          = MaterialTheme.colorScheme.primaryContainer,
                tonalElevation = 4.dp,
                modifier       = Modifier
                    .size(28.dp)
                    .clickable { launcher.launch("image/*") },
            ) {
                Icon(
                    imageVector        = Icons.Default.CameraAlt,
                    contentDescription = "Cambiar avatar",
                    tint               = MaterialTheme.colorScheme.onPrimaryContainer,
                    modifier           = Modifier.padding(4.dp).fillMaxSize(),
                )
            }
        }
    }
}

@Composable
private fun InitialsAvatar(username: String) {
    Box(
        contentAlignment = Alignment.Center,
        modifier         = Modifier
            .fillMaxSize()
            .background(avatarBackground(username)),
    ) {
        Text(
            text       = username.take(2).uppercase(),
            color      = Color.White,
            fontSize   = 32.sp,
            fontWeight = FontWeight.Bold,
        )
    }
}

private fun avatarBackground(username: String): Color {
    val hue = (username.hashCode().and(0x7FFFFFFF) % 360).toFloat()
    return Color.hsl(hue = hue, saturation = 0.55f, lightness = 0.45f)
}
```

---

## 12.14 `ProfileScreen` — integrar `AvatarSection`

### `presentation/ui/client/profile/ProfileScreen.kt`

Reemplaza la `ProfileScreen` existente. Usa `ProfileViewModel` (via `hiltViewModel()`) en lugar de `AuthViewModel` para cargar los datos del perfil.

`presentation/ui/client/profile/ProfileScreen.kt`
```kotlin
package com.shopapp.presentation.ui.client.profile

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.Logout
import androidx.compose.material.icons.filled.Edit
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.presentation.viewmodel.ProfileViewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ProfileScreen(
    onEditProfile: () -> Unit       = {},
    onLogout:      () -> Unit       = {},
    viewModel:     ProfileViewModel = hiltViewModel(),
) {
    val state by viewModel.state.collectAsState()
    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(Unit) {
        viewModel.snackbar.collect { message ->
            snackbarHostState.showSnackbar(message)
        }
    }

    Scaffold(
        topBar        = { TopAppBar(title = { Text("Mi perfil") }) },
        snackbarHost  = { SnackbarHost(snackbarHostState) },
    ) { padding ->
        when {
            state.isLoading -> {
                Box(
                    contentAlignment = Alignment.Center,
                    modifier         = Modifier.fillMaxSize().padding(padding),
                ) { CircularProgressIndicator() }
            }

            state.error != null -> {
                Box(
                    contentAlignment = Alignment.Center,
                    modifier         = Modifier.fillMaxSize().padding(padding),
                ) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Text(state.error ?: "Error", color = MaterialTheme.colorScheme.error)
                        Spacer(Modifier.height(8.dp))
                        Button(onClick = { viewModel.loadProfile() }) { Text("Reintentar") }
                    }
                }
            }

            else -> {
                val profile = state.profile

                Column(
                    horizontalAlignment = Alignment.CenterHorizontally,
                    modifier            = Modifier
                        .fillMaxSize()
                        .padding(padding)
                        .verticalScroll(rememberScrollState())
                        .padding(horizontal = 16.dp),
                ) {
                    Spacer(Modifier.height(24.dp))

                    // ── Avatar ───────────────────────────────────────────────
                    AvatarSection(
                        avatarUrl       = state.avatarUrl,
                        username        = profile?.username ?: "",
                        isUploading     = state.isUploading,
                        onImageSelected = { uri -> viewModel.uploadAvatar(uri) },
                    )

                    Spacer(Modifier.height(12.dp))

                    Text(
                        text       = profile?.username ?: "—",
                        style      = MaterialTheme.typography.titleLarge,
                        fontWeight = FontWeight.Bold,
                    )
                    Text(
                        text  = profile?.email ?: "—",
                        style = MaterialTheme.typography.bodyMedium,
                        color = MaterialTheme.colorScheme.onSurfaceVariant,
                    )

                    Spacer(Modifier.height(4.dp))

                    if (profile?.isStaff == true) {
                        SuggestionChip(onClick = {}, label = { Text("Staff") })
                    }

                    Spacer(Modifier.height(24.dp))
                    HorizontalDivider()
                    Spacer(Modifier.height(16.dp))

                    OutlinedButton(
                        onClick  = onEditProfile,
                        modifier = Modifier.fillMaxWidth(),
                    ) {
                        Icon(Icons.Default.Edit, null, modifier = Modifier.size(18.dp))
                        Spacer(Modifier.width(8.dp))
                        Text("Editar perfil")
                    }

                    Spacer(Modifier.height(8.dp))

                    OutlinedButton(
                        onClick  = onLogout,
                        modifier = Modifier.fillMaxWidth(),
                        colors   = ButtonDefaults.outlinedButtonColors(
                            contentColor = MaterialTheme.colorScheme.error,
                        ),
                    ) {
                        Icon(
                            Icons.AutoMirrored.Filled.Logout,
                            null,
                            modifier = Modifier.size(18.dp),
                        )
                        Spacer(Modifier.width(8.dp))
                        Text("Cerrar sesión")
                    }

                    Spacer(Modifier.height(24.dp))
                }
            }
        }
    }
}
```

---

## 12.15 Actualizar `NavGraph.kt` — nueva firma de `ProfileScreen`

La nueva `ProfileScreen` ya no recibe `authViewModel` como parámetro (lo obtiene internamente via `ProfileViewModel`). Actualiza el `composable(Screen.Profile.route)` en `NavGraph.kt`:

`presentation/navigation/NavGraph.kt  (actualizar composable existente)`
```kotlin

composable(Screen.Profile.route) {
    if (!isAuthenticated) {
        LaunchedEffect(Unit) {
            navController.navigate(Screen.Login.route) {
                popUpTo(Screen.Home.route)
            }
        }
    } else {
        ProfileScreen(
            onLogout = {
                authViewModel.logout()
                navController.navigate(Screen.Login.route) {
                    popUpTo(0) { inclusive = true }
                }
            },
        )
    }
}
```

> `authViewModel.logout()` sigue siendo necesario en el callback para limpiar el estado de sesión; lo que cambia es que `ProfileScreen` ya no recibe `authViewModel` como parámetro.

---

## 🚦 PARADA 2 — Avatar de usuario

> **Qué está listo:** campo `avatarUrl` en `UserDto` y `User`, endpoints `getProfile` y `uploadAvatar` en `UserApi`, métodos en `UserRepository`, `ProfileViewModel`, `AvatarSection`, `ProfileScreen` actualizada, `NavGraph` actualizado.
> **Qué vas a probar:**

**Prueba 1 — Ver perfil con iniciales**

1. Autenticarse como cualquier usuario.
2. Ir a **Perfil** desde la barra de navegación inferior.
3. En la parte superior debe aparecer el avatar circular con las iniciales del username sobre un fondo de color.

**Prueba 2 — Subir avatar**

1. Pulsar el avatar o el ícono de cámara 📷.
2. Seleccionar una imagen JPEG o PNG de la galería.
3. Resultado esperado:
   - Spinner en el avatar mientras se sube.
   - Snackbar: **"Avatar actualizado correctamente"**.
   - El avatar muestra la nueva foto sin recargar la pantalla.

**Prueba 3 — Persistencia**

1. Cerrar sesión y volver a iniciar sesión.
2. Ir a **Perfil** → el avatar debe mostrar la foto subida anteriormente.

**Casos de borde:**

| Acción | Resultado esperado |
|--------|--------------------|
| Imagen mayor de 2 MB | Error `400` del backend mostrado en Snackbar |
| Formato no soportado | Error `400` del backend mostrado en Snackbar |
| Sin conexión | Snackbar con el mensaje de error de red |

---

## Permisos en `AndroidManifest.xml`

No se necesitan permisos adicionales en Android 13+ para acceder a imágenes mediante `ActivityResultContracts.GetContent()`. En versiones anteriores (API 32 o menor) se puede declarar `READ_EXTERNAL_STORAGE` con `maxSdkVersion`:

```xml
<!-- AndroidManifest.xml — opcional para API 32 o menor -->
<!--
<uses-permission
    android:name="android.permission.READ_EXTERNAL_STORAGE"
    android:maxSdkVersion="32" />
-->
```

---

## Checkpoint Módulo 12

| # | Verificación | Resultado esperado |
|---|---|---|
| 1 | Sincronizar Gradle tras añadir `coil-compose` | Build exitoso |
| 2 | Abrir `ProductDetailScreen` desde el catálogo | Imagen cargada con Coil en banner 320 dp |
| 3 | Producto sin imagen en `ProductDetailScreen` | Se muestra el emoji 📦 como placeholder |
| 4 | Pulsar lápiz en un producto como staff | Sheet de edición se abre con imagen en la parte superior |
| 5 | Subir imagen de producto como staff | Snackbar de éxito, lista se recarga con nueva imagen |
| 6 | Intentar subir un archivo PDF | Error `400` del backend en Snackbar |
| 7 | Abrir `ProfileScreen` como cliente | Avatar circular con iniciales del username |
| 8 | Pulsar avatar y seleccionar una imagen | Spinner durante la subida, nueva foto al terminar |
| 9 | Cerrar y reabrir la app | Avatar persiste (URL almacenada en el backend) |

---

> **Nota sobre el backend (shopapi_07):** `image_url` en `ProductSerializer` y `avatar_url` en `UserProfileSerializer` devuelven `null` si el modelo no tiene archivo asociado. El servidor valida tamaño máximo de **2 MB** y tipos MIME **JPEG / PNG / WebP**; cualquier violación devuelve `400 Bad Request` con el detalle del error en el cuerpo JSON, que Retrofit convierte en una excepción capturada por `runCatching`.
