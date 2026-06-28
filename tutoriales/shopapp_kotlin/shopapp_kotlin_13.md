# ShopApp Android — Módulo 13
## Correo electrónico: recuperación de contraseña y notificaciones

> **Objetivo:** Integrar en la app dos flujos relacionados con el servicio de email de ShopAPI: la recuperación de contraseña (dos pantallas) y el envío de notificaciones manuales (solo staff).
> **Checkpoint final:** el usuario puede solicitar y confirmar un reset de contraseña desde la app; el staff puede enviar correos individuales o masivos desde una pantalla dedicada.

### ¿Qué necesita implementación en la app?

De los cuatro flujos de correo implementados en **ShopAPI — Etapa 8**, solo dos requieren pantallas nuevas:

| Flujo | Quién lo dispara | ¿Necesita pantalla? |
|-------|-----------------|---------------------|
| Correo de bienvenida | Backend (señal `post_save` de `User`) | ❌ No — ocurre al registrar |
| Recuperación de contraseña | El usuario solicita el reset desde la app | ✅ Sí — 2 pantallas |
| Confirmación de orden | Backend (señal `post_save` de `Order`) | ❌ No — ocurre al crear la orden |
| Notificaciones manuales | Staff desde la app | ✅ Sí — 1 pantalla |

El registro y la creación de órdenes ya existen en módulos anteriores. Los correos de bienvenida y confirmación de orden se disparan automáticamente en el backend mediante señales Django, sin ninguna intervención de la app.

| Parada | Qué se prueba |
|--------|--------------|
| 🚦 1A | Solicitar reset → llega correo a Gmail con uid y token |
| 🚦 1B | Confirmar reset con uid + token → login con nueva contraseña |
| 🚦 2 | Envío individual y masivo como staff — usuario normal ve 403 |

---

## ── Parte 1 · Recuperación de contraseña ────────────────────────────────────

El flujo tiene dos pasos:

1. **Solicitar el reset** (`ForgotPasswordScreen`): el usuario ingresa su email → el backend envía un correo con un enlace que contiene `uid` y `token`.
2. **Confirmar el reset** (`ResetPasswordConfirmScreen`): el usuario copia el `uid` y `token` del enlace del correo → ingresa la nueva contraseña → la app navega al login.

> **Nota sobre deep links:** En producción se configura un *deep link* para que el enlace del correo abra directamente `ResetPasswordConfirmScreen` con el `uid` y `token` precargados. En este módulo el usuario los copia manualmente, lo cual es suficiente para validar el flujo completo de la API sin configuración adicional de Intent Filters o Android App Links.

---

## 13.1 DTOs de recuperación de contraseña

Añade al final de `data/remote/dto/AuthDto.kt`:

`data/remote/dto/AuthDto.kt  (añadir al final)`
```kotlin
import com.google.gson.annotations.SerializedName

/** Cuerpo del POST /api/auth/password-reset/ */
data class PasswordResetRequestDto(
    @SerializedName("email") val email: String,
)

/** Cuerpo del POST /api/auth/password-reset/confirm/ */
data class PasswordResetConfirmDto(
    @SerializedName("uid")           val uid:          String,
    @SerializedName("token")         val token:        String,
    @SerializedName("new_password")  val newPassword:  String,
    @SerializedName("new_password2") val newPassword2: String,
)

/**
 * Respuesta genérica { "detail": "..." }
 * Usada por ambos endpoints de recuperación de contraseña.
 */
data class MessageDto(
    @SerializedName("detail") val detail: String,
)
```

---

## 13.2 Endpoints en `AuthApi.kt`

Los endpoints de recuperación de contraseña usan la ruta `/api/auth/...`, por lo que pertenecen en `AuthApi` junto con login, register y logout.

> **Si el proyecto ya tiene estos endpoints en `UserApi.kt`** (de una sesión anterior), elimínalos de allí antes de continuar.

Añade al final de la interfaz existente:

`data/remote/api/AuthApi.kt  (añadir al final)`
```kotlin
import com.shopapp.data.remote.dto.MessageDto
import com.shopapp.data.remote.dto.PasswordResetConfirmDto
import com.shopapp.data.remote.dto.PasswordResetRequestDto
import retrofit2.Response
import retrofit2.http.Body
import retrofit2.http.POST

    // ── Recuperación de contraseña ───────────────────────────────────────────

    /** Backend: POST /api/auth/password-reset/ — no requiere autenticación */
    @POST("auth/password-reset/")
    suspend fun requestPasswordReset(
        @Body body: PasswordResetRequestDto,
    ): Response<MessageDto>

    /** Backend: POST /api/auth/password-reset/confirm/ */
    @POST("auth/password-reset/confirm/")
    suspend fun confirmPasswordReset(
        @Body body: PasswordResetConfirmDto,
    ): Response<MessageDto>
```

> **Interceptor de autenticación:** estos endpoints son públicos. El `BearerTokenInterceptor` solo añade el header cuando el token no es nulo, por lo que no hay problema si el usuario no está autenticado.

---

## 13.3 Extender `AuthRepository` y `AuthRepositoryImpl`

El reset de contraseña es un flujo de autenticación: va en `AuthRepository` junto con login, register y logout. `AuthRepositoryImpl` ya inyecta `AuthApi`, así que no se necesita ningún nuevo binding de DI.

### Interfaz — `domain/repository/AuthRepository.kt`

Añade al final de la interfaz:

`domain/repository/AuthRepository.kt  (añadir al final)`
```kotlin
    // ── Recuperación de contraseña ───────────────────────────────────────────
    suspend fun requestReset(email: String): Result<String>
    suspend fun confirmReset(
        uid:          String,
        token:        String,
        newPassword:  String,
        newPassword2: String,
    ): Result<String>
```

### Implementación — `data/repository/AuthRepositoryImpl.kt`

Añade los imports necesarios y los métodos al final de la clase, antes del `}` de cierre:

`data/repository/AuthRepositoryImpl.kt  (añadir al final)`
```kotlin
import com.shopapp.data.remote.dto.PasswordResetConfirmDto
import com.shopapp.data.remote.dto.PasswordResetRequestDto

    override suspend fun requestReset(email: String): Result<String> =
        runCatching {
            val response = api.requestPasswordReset(PasswordResetRequestDto(email))
            if (response.isSuccessful) {
                response.body()?.detail ?: "Solicitud enviada"
            } else {
                error(response.errorBody()?.string() ?: "Error ${response.code()}")
            }
        }

    override suspend fun confirmReset(
        uid:          String,
        token:        String,
        newPassword:  String,
        newPassword2: String,
    ): Result<String> =
        runCatching {
            val response = api.confirmPasswordReset(
                PasswordResetConfirmDto(
                    uid          = uid,
                    token        = token,
                    newPassword  = newPassword,
                    newPassword2 = newPassword2,
                )
            )
            if (response.isSuccessful) {
                response.body()?.detail ?: "Contraseña actualizada"
            } else {
                error(response.errorBody()?.string() ?: "Error ${response.code()}")
            }
        }
```

> **Sin DI adicional:** `AuthRepository` ya está enlazado en `RepositoryModule`. No se necesita ningún cambio en el módulo de inyección de dependencias.

---

## 13.4 ViewModels de recuperación de contraseña

### `ForgotPasswordViewModel` — `presentation/viewmodel/ForgotPasswordViewModel.kt`

`presentation/viewmodel/ForgotPasswordViewModel.kt`
```kotlin
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.repository.AuthRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

data class ForgotPasswordUiState(
    val isLoading: Boolean = false,
    val emailSent: Boolean = false,   // true tras respuesta 200 del backend
    val error:     String? = null,
)

@HiltViewModel
class ForgotPasswordViewModel @Inject constructor(
    private val repository: AuthRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(ForgotPasswordUiState())
    val state: StateFlow<ForgotPasswordUiState> = _state.asStateFlow()

    fun requestReset(email: String) {
        if (_state.value.isLoading) return
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            repository.requestReset(email.trim())
                .onSuccess {
                    _state.update { it.copy(isLoading = false, emailSent = true) }
                }
                .onFailure { e ->
                    _state.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    fun clearError() {
        _state.update { it.copy(error = null) }
    }
}
```

### `ResetPasswordConfirmViewModel` — `presentation/viewmodel/ResetPasswordConfirmViewModel.kt`

`presentation/viewmodel/ResetPasswordConfirmViewModel.kt`
```kotlin
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.repository.AuthRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

data class ResetPasswordConfirmUiState(
    val isLoading:    Boolean = false,
    val resetSuccess: Boolean = false,  // true tras 200 OK del backend
    val error:        String? = null,
)

@HiltViewModel
class ResetPasswordConfirmViewModel @Inject constructor(
    private val repository: AuthRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(ResetPasswordConfirmUiState())
    val state: StateFlow<ResetPasswordConfirmUiState> = _state.asStateFlow()

    fun confirmReset(
        uid:          String,
        token:        String,
        newPassword:  String,
        newPassword2: String,
    ) {
        if (_state.value.isLoading) return
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            repository.confirmReset(uid.trim(), token.trim(), newPassword, newPassword2)
                .onSuccess {
                    _state.update { it.copy(isLoading = false, resetSuccess = true) }
                }
                .onFailure { e ->
                    _state.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    fun clearError() {
        _state.update { it.copy(error = null) }
    }
}
```

---

## 13.5 `ForgotPasswordScreen`

### `presentation/ui/auth/ForgotPasswordScreen.kt`

`presentation/ui/auth/ForgotPasswordScreen.kt`
```kotlin
package com.shopapp.presentation.ui.auth

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ArrowBack
import androidx.compose.material.icons.filled.Email
import androidx.compose.material.icons.filled.MarkEmailRead
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.presentation.viewmodel.ForgotPasswordViewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ForgotPasswordScreen(
    onBack:        () -> Unit,
    onGoToConfirm: () -> Unit,
    viewModel:     ForgotPasswordViewModel = hiltViewModel(),
) {
    val state by viewModel.state.collectAsState()
    var email by remember { mutableStateOf("") }

    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(state.error) {
        state.error?.let {
            snackbarHostState.showSnackbar(it)
            viewModel.clearError()
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Recuperar contraseña") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(
                            imageVector        = Icons.AutoMirrored.Filled.ArrowBack,
                            contentDescription = "Volver",
                        )
                    }
                },
            )
        },
        snackbarHost = { SnackbarHost(snackbarHostState) },
    ) { padding ->
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(horizontal = 24.dp),
        ) {
            Spacer(Modifier.height(48.dp))

            if (!state.emailSent) {
                // ── Formulario de solicitud ───────────────────────────────────
                Icon(
                    imageVector        = Icons.Default.Email,
                    contentDescription = null,
                    tint               = MaterialTheme.colorScheme.primary,
                    modifier           = Modifier.size(64.dp),
                )

                Spacer(Modifier.height(24.dp))

                Text(
                    text      = "Ingresa tu correo electrónico y te enviaremos un enlace para restablecer tu contraseña.",
                    textAlign = TextAlign.Center,
                    style     = MaterialTheme.typography.bodyMedium,
                    color     = MaterialTheme.colorScheme.onSurfaceVariant,
                )

                Spacer(Modifier.height(32.dp))

                OutlinedTextField(
                    value           = email,
                    onValueChange   = { email = it },
                    label           = { Text("Correo electrónico") },
                    singleLine      = true,
                    enabled         = !state.isLoading,
                    keyboardOptions = KeyboardOptions(
                        keyboardType = KeyboardType.Email,
                        imeAction    = ImeAction.Done,
                    ),
                    modifier = Modifier.fillMaxWidth(),
                )

                Spacer(Modifier.height(24.dp))

                Button(
                    onClick  = { viewModel.requestReset(email) },
                    enabled  = email.isNotBlank() && !state.isLoading,
                    modifier = Modifier.fillMaxWidth(),
                ) {
                    if (state.isLoading) {
                        CircularProgressIndicator(
                            modifier    = Modifier.size(18.dp),
                            color       = MaterialTheme.colorScheme.onPrimary,
                            strokeWidth = 2.dp,
                        )
                        Spacer(Modifier.width(8.dp))
                    }
                    Text(if (state.isLoading) "Enviando..." else "Enviar enlace")
                }

            } else {
                // ── Confirmación de envío ─────────────────────────────────────
                Icon(
                    imageVector        = Icons.Default.MarkEmailRead,
                    contentDescription = null,
                    tint               = MaterialTheme.colorScheme.primary,
                    modifier           = Modifier.size(64.dp),
                )

                Spacer(Modifier.height(24.dp))

                Text(
                    text       = "Revisa tu correo",
                    style      = MaterialTheme.typography.headlineSmall,
                    fontWeight = FontWeight.Bold,
                )

                Spacer(Modifier.height(12.dp))

                Text(
                    text      = "Si el correo está registrado, recibirás el enlace en unos minutos.\n\nAbre el enlace del correo, copia el uid y el token, y úsalos en el siguiente paso.",
                    textAlign = TextAlign.Center,
                    style     = MaterialTheme.typography.bodyMedium,
                    color     = MaterialTheme.colorScheme.onSurfaceVariant,
                )

                Spacer(Modifier.height(32.dp))

                Button(
                    onClick  = onGoToConfirm,
                    modifier = Modifier.fillMaxWidth(),
                ) {
                    Text("Tengo el código → Restablecer contraseña")
                }

                Spacer(Modifier.height(8.dp))

                TextButton(onClick = onBack) {
                    Text("Volver al inicio de sesión")
                }
            }
        }
    }
}
```

> **Flujo de la pantalla:** el mismo Composable maneja dos estados visuales. Cuando `emailSent = false` muestra el formulario; cuando `emailSent = true` muestra el mensaje de confirmación con el botón para continuar al paso 2. Esto evita una navegación intermedia y mantiene el ViewModel vivo entre ambos estados.

---

## 13.6 `ResetPasswordConfirmScreen`

### `presentation/ui/auth/ResetPasswordConfirmScreen.kt`

`presentation/ui/auth/ResetPasswordConfirmScreen.kt`
```kotlin
package com.shopapp.presentation.ui.auth

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ArrowBack
import androidx.compose.material.icons.filled.Lock
import androidx.compose.material.icons.filled.Visibility
import androidx.compose.material.icons.filled.VisibilityOff
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.text.input.PasswordVisualTransformation
import androidx.compose.ui.text.input.VisualTransformation
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.presentation.viewmodel.ResetPasswordConfirmViewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ResetPasswordConfirmScreen(
    onBack:         () -> Unit,
    onResetSuccess: () -> Unit,             // navega al login tras reset exitoso
    viewModel:      ResetPasswordConfirmViewModel = hiltViewModel(),
) {
    val state by viewModel.state.collectAsState()
    var uid          by remember { mutableStateOf("") }
    var token        by remember { mutableStateOf("") }
    var newPassword  by remember { mutableStateOf("") }
    var newPassword2 by remember { mutableStateOf("") }
    var showPass     by remember { mutableStateOf(false) }

    val snackbarHostState = remember { SnackbarHostState() }

    // Navegar al login tras reset exitoso
    LaunchedEffect(state.resetSuccess) {
        if (state.resetSuccess) {
            snackbarHostState.showSnackbar("Contraseña actualizada. Inicia sesión.")
            onResetSuccess()
        }
    }

    // Mostrar errores del servidor
    LaunchedEffect(state.error) {
        state.error?.let {
            snackbarHostState.showSnackbar(it)
            viewModel.clearError()
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Nueva contraseña") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(
                            imageVector        = Icons.AutoMirrored.Filled.ArrowBack,
                            contentDescription = "Volver",
                        )
                    }
                },
            )
        },
        snackbarHost = { SnackbarHost(snackbarHostState) },
    ) { padding ->
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(horizontal = 24.dp)
                .verticalScroll(rememberScrollState()),
        ) {
            Spacer(Modifier.height(24.dp))

            Icon(
                imageVector        = Icons.Default.Lock,
                contentDescription = null,
                tint               = MaterialTheme.colorScheme.primary,
                modifier           = Modifier.size(48.dp),
            )

            Spacer(Modifier.height(12.dp))

            Text(
                text  = "Pega el uid y el token del enlace que recibiste por correo y elige una nueva contraseña.",
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant,
            )

            Spacer(Modifier.height(24.dp))

            // ── UID ───────────────────────────────────────────────────────────
            OutlinedTextField(
                value           = uid,
                onValueChange   = { uid = it },
                label           = { Text("UID") },
                placeholder     = { Text("ej. MQ") },
                singleLine      = true,
                enabled         = !state.isLoading,
                keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
                modifier        = Modifier.fillMaxWidth(),
            )

            Spacer(Modifier.height(8.dp))

            // ── Token ─────────────────────────────────────────────────────────
            OutlinedTextField(
                value           = token,
                onValueChange   = { token = it },
                label           = { Text("Token") },
                placeholder     = { Text("ej. abc-defg-hij") },
                singleLine      = true,
                enabled         = !state.isLoading,
                keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
                modifier        = Modifier.fillMaxWidth(),
            )

            Spacer(Modifier.height(8.dp))

            // ── Nueva contraseña ──────────────────────────────────────────────
            OutlinedTextField(
                value                = newPassword,
                onValueChange        = { newPassword = it },
                label                = { Text("Nueva contraseña") },
                singleLine           = true,
                enabled              = !state.isLoading,
                visualTransformation = if (showPass) VisualTransformation.None
                                       else PasswordVisualTransformation(),
                trailingIcon = {
                    IconButton(onClick = { showPass = !showPass }) {
                        Icon(
                            imageVector        = if (showPass) Icons.Default.VisibilityOff
                                                else Icons.Default.Visibility,
                            contentDescription = if (showPass) "Ocultar contraseña"
                                                else "Mostrar contraseña",
                        )
                    }
                },
                keyboardOptions = KeyboardOptions(
                    keyboardType = KeyboardType.Password,
                    imeAction    = ImeAction.Next,
                ),
                modifier = Modifier.fillMaxWidth(),
            )

            Spacer(Modifier.height(8.dp))

            // ── Confirmar nueva contraseña ────────────────────────────────────
            val passwordMismatch = newPassword2.isNotEmpty() && newPassword != newPassword2

            OutlinedTextField(
                value                = newPassword2,
                onValueChange        = { newPassword2 = it },
                label                = { Text("Confirmar contraseña") },
                singleLine           = true,
                enabled              = !state.isLoading,
                visualTransformation = if (showPass) VisualTransformation.None
                                       else PasswordVisualTransformation(),
                isError              = passwordMismatch,
                supportingText       = {
                    if (passwordMismatch) Text("Las contraseñas no coinciden")
                },
                keyboardOptions = KeyboardOptions(
                    keyboardType = KeyboardType.Password,
                    imeAction    = ImeAction.Done,
                ),
                modifier = Modifier.fillMaxWidth(),
            )

            Spacer(Modifier.height(24.dp))

            val isFormValid = uid.isNotBlank() && token.isNotBlank()
                && newPassword.isNotBlank() && newPassword == newPassword2

            Button(
                onClick  = {
                    viewModel.confirmReset(uid, token, newPassword, newPassword2)
                },
                enabled  = isFormValid && !state.isLoading,
                modifier = Modifier.fillMaxWidth(),
            ) {
                if (state.isLoading) {
                    CircularProgressIndicator(
                        modifier    = Modifier.size(18.dp),
                        color       = MaterialTheme.colorScheme.onPrimary,
                        strokeWidth = 2.dp,
                    )
                    Spacer(Modifier.width(8.dp))
                }
                Text(if (state.isLoading) "Guardando..." else "Restablecer contraseña")
            }

            Spacer(Modifier.height(16.dp))
        }
    }
}
```

> **Validación en UI:** el botón está deshabilitado mientras las contraseñas no coincidan. Si el token ya fue usado, el backend devuelve `400` con el detalle del error, que `runCatching` convierte en `Result.failure` y se muestra en el Snackbar.

---

## 13.7 Actualizar `LoginScreen` — agregar enlace "¿Olvidaste tu contraseña?"

Añade el parámetro `onForgotPassword` a la firma del Composable y agrega el `TextButton` debajo del botón principal de login:

`presentation/ui/auth/LoginScreen.kt  — fragmento relevante`
```kotlin

@Composable
fun LoginScreen(
    onLoginSuccess:       (isStaff: Boolean) -> Unit,
    onNavigateToRegister: () -> Unit,
    onForgotPassword:     () -> Unit = {},   // ← nuevo parámetro
    viewModel:            AuthViewModel = hiltViewModel(),
) {
    // ... código existente ...

    // DENTRO del Column de formulario, después del Button de login:
    TextButton(
        onClick  = onForgotPassword,
        modifier = Modifier.align(Alignment.CenterHorizontally),
    ) {
        Text("¿Olvidaste tu contraseña?")
    }
}
```

---

## 13.8 Actualizar la navegación

Primero añade las rutas nuevas a `Screen.kt`:

`presentation/navigation/Screen.kt  (añadir en el bloque Auth)`
```kotlin

data object ForgotPassword       : Screen("forgot-password")
data object ResetPasswordConfirm : Screen("reset-password-confirm")
```

Luego añade las dos rutas nuevas al `NavHost` existente en `NavGraph.kt`:

`presentation/navigation/NavGraph.kt  (añadir dentro del NavHost)`
```kotlin

// ── Recuperación de contraseña ───────────────────────────────────────────────

composable(Screen.ForgotPassword.route) {
    ForgotPasswordScreen(
        onBack        = { navController.popBackStack() },
        onGoToConfirm = { navController.navigate(Screen.ResetPasswordConfirm.route) },
    )
}

composable(Screen.ResetPasswordConfirm.route) {
    ResetPasswordConfirmScreen(
        onBack         = { navController.popBackStack() },
        onResetSuccess = {
            navController.navigate(Screen.Login.route) {
                popUpTo(Screen.Login.route) { inclusive = true }
            }
        },
    )
}
```

Actualiza la llamada a `LoginScreen` para conectar el nuevo enlace:

`presentation/navigation/NavGraph.kt  (actualizar composable existente)`
```kotlin

composable(Screen.Login.route) {
    LoginScreen(
        onLoginSuccess       = { staff ->
            val dest = if (staff) Screen.AdminDashboard.route else Screen.Home.route
            navController.navigate(dest) {
                popUpTo(Screen.Login.route) { inclusive = true }
            }
        },
        onNavigateToRegister = { navController.navigate(Screen.Register.route) },
        onForgotPassword     = { navController.navigate(Screen.ForgotPassword.route) },  // ← nuevo
        viewModel            = authViewModel,
    )
}
```

---

## 🚦 PARADA 1A — Solicitar reset de contraseña

> **Qué está listo:** DTOs añadidos a `AuthDto.kt`, endpoints en `AuthApi`, métodos en `AuthRepository`, `ForgotPasswordViewModel`, `ForgotPasswordScreen`, enlace "¿Olvidaste tu contraseña?" en `LoginScreen`, rutas en el NavHost.
> **Qué vas a probar:** que al ingresar el email el backend envía el correo con el enlace de reset.

1. Construir y ejecutar la app.
2. En `LoginScreen` pulsar **"¿Olvidaste tu contraseña?"** → navega a `ForgotPasswordScreen`.
3. Ingresar el email de un usuario registrado.
4. Pulsar **"Enviar enlace"** → el botón muestra "Enviando..." mientras la petición está en vuelo.
5. Tras la respuesta `200 OK` la pantalla cambia al estado de confirmación con el icono de email.
6. Abrir Gmail → debe haber llegado un correo con asunto **"Recuperación de contraseña — ShopAPI"**.
7. Copiar los valores de `uid` y `token` del enlace del botón:
   ```
   http://localhost:3000/password-reset/confirm/?uid=MQ&token=abc-defg-hij
   ```
   Los valores a copiar son `uid=MQ` y `token=abc-defg-hij` (los tuyos serán distintos).

> **Caso de borde:** ingresa un email que no existe → la app muestra igualmente la pantalla de confirmación. El backend siempre responde `200` para evitar revelar qué cuentas están registradas.

---

## 🚦 PARADA 1B — Confirmar reset y login con nueva contraseña

> **Qué está listo:** `ResetPasswordConfirmViewModel`, `ResetPasswordConfirmScreen`, ruta en el NavHost.
> **Qué vas a probar:** que con el uid y token del correo se puede cambiar la contraseña y hacer login con las nuevas credenciales.

1. Desde la pantalla "Revisa tu correo" pulsar **"Tengo el código → Restablecer contraseña"** → navega a `ResetPasswordConfirmScreen`.
2. Pegar el `uid` y `token` copiados del correo en sus respectivos campos.
3. Ingresar la nueva contraseña en ambos campos (mínimo 8 caracteres, deben coincidir).
4. El botón se habilita solo cuando todos los campos están rellenos y las contraseñas coinciden.
5. Pulsar **"Restablecer contraseña"** → respuesta esperada `200 OK`.
6. La app navega automáticamente a `LoginScreen` y muestra el Snackbar "Contraseña actualizada. Inicia sesión."
7. Iniciar sesión con la nueva contraseña → `200 OK` con nuevos tokens JWT.

**Casos de borde:**

| Acción | Resultado esperado |
|--------|--------------------|
| Token incorrecto o ya usado | Error en Snackbar: "Token inválido o expirado" |
| UID inválido (inventado) | Error en Snackbar: "Enlace inválido o expirado" |
| Contraseñas que no coinciden | Botón deshabilitado, texto de error bajo el campo |
| Contraseña menor de 8 caracteres | Error `400` del backend mostrado en Snackbar |

---

## ── Parte 2 · Notificaciones de staff ────────────────────────────────────────

Un miembro del staff puede enviar un correo personalizado a un usuario específico o, sin indicar destinatario, enviarlo masivamente a todos los usuarios activos no-staff. Esta pantalla solo es accesible cuando `isStaff = true`.

---

## 13.9 DTOs de notificación

Añade al final de `data/remote/dto/UserDto.kt`:

`data/remote/dto/UserDto.kt  (añadir al final)`
```kotlin
import com.google.gson.annotations.SerializedName

/** Cuerpo del POST /api/emails/send/ */
data class SendNotificationDto(
    @SerializedName("subject") val subject: String,
    @SerializedName("message") val message: String,
    @SerializedName("user_id") val userId:  Int? = null,  // null → envío masivo
)

/**
 * Respuesta { "detail": "Correo enviado a N usuario(s).", "sent": N, "failed": M }
 */
data class NotificationResultDto(
    @SerializedName("detail") val detail: String,
    @SerializedName("sent")   val sent:   Int,
    @SerializedName("failed") val failed: Int,
)
```

---

## 13.10 Endpoint en `UserApi.kt`

Añade al final de la interfaz:

`data/remote/api/UserApi.kt`
```kotlin
import com.shopapp.data.remote.dto.NotificationResultDto
import com.shopapp.data.remote.dto.SendNotificationDto
import retrofit2.Response
import retrofit2.http.Body
import retrofit2.http.POST

interface UserApi {
    // ... endpoints existentes y de reset (sección 13.2) ...

    // ── Notificaciones de staff ───────────────────────────────────────────────

    /**
     * Envía un correo personalizado o masivo.
     * Requiere is_staff = true en el backend (IsAdminUser → 403 si no es staff).
     * Backend: POST /api/emails/send/
     */
    @POST("emails/send/")
    suspend fun sendNotification(
        @Body body: SendNotificationDto,
    ): Response<NotificationResultDto>
}
```

---

## 13.11 Modelo de dominio y extender `UserRepository`

### Modelo — `domain/model/NotificationResult.kt`

Crea el archivo de modelo de dominio:

`domain/model/NotificationResult.kt`
```kotlin
package com.shopapp.domain.model

data class NotificationResult(
    val detail: String,
    val sent:   Int,
    val failed: Int,
)
```

### Interfaz — `domain/repository/UserRepository.kt`

Añade al final de la interfaz:

`domain/repository/UserRepository.kt  (añadir al final)`
```kotlin
    // ── Notificaciones de staff ───────────────────────────────────────────────
    suspend fun sendNotification(
        subject: String,
        message: String,
        userId:  Int? = null,
    ): Result<NotificationResult>
```

### Implementación — `data/repository/UserRepositoryImpl.kt`

Añade el import al bloque de imports y el método al final de la clase:

`data/repository/UserRepositoryImpl.kt  (añadir al final)`
```kotlin
import com.shopapp.data.remote.dto.SendNotificationDto
import com.shopapp.domain.model.NotificationResult

    override suspend fun sendNotification(
        subject: String,
        message: String,
        userId:  Int?,
    ): Result<NotificationResult> =
        runCatching {
            val response = api.sendNotification(SendNotificationDto(subject, message, userId))
            if (response.isSuccessful) {
                val dto = response.body() ?: error("Respuesta vacía del servidor")
                NotificationResult(dto.detail, dto.sent, dto.failed)
            } else {
                error(response.errorBody()?.string() ?: "Error ${response.code()}")
            }
        }
```

> **Sin DI adicional:** `UserRepository` ya está enlazado en `RepositoryModule`. No se necesita ningún cambio en el módulo de inyección de dependencias.

---

## 13.12 `SendNotificationViewModel`

### `presentation/viewmodel/SendNotificationViewModel.kt`

`presentation/viewmodel/SendNotificationViewModel.kt`
```kotlin
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.NotificationResult
import com.shopapp.domain.repository.UserRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

data class SendNotificationUiState(
    val isLoading: Boolean             = false,
    val result:    NotificationResult? = null,  // resultado del último envío
    val error:     String?             = null,
)

@HiltViewModel
class SendNotificationViewModel @Inject constructor(
    private val repository: UserRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(SendNotificationUiState())
    val state: StateFlow<SendNotificationUiState> = _state.asStateFlow()

    fun send(subject: String, message: String, userId: Int?) {
        if (_state.value.isLoading) return
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null, result = null) }
            repository.sendNotification(subject, message, userId)
                .onSuccess { result ->
                    _state.update { it.copy(isLoading = false, result = result) }
                }
                .onFailure { e ->
                    _state.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    fun clearResult() { _state.update { it.copy(result = null) } }
    fun clearError()  { _state.update { it.copy(error = null) } }
}
```

---

## 13.13 `SendNotificationScreen`

### `presentation/ui/admin/users/SendNotificationScreen.kt`

`presentation/ui/admin/users/SendNotificationScreen.kt`
```kotlin
package com.shopapp.presentation.ui.admin.users

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.automirrored.filled.ArrowBack
import androidx.compose.material.icons.filled.Send
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.presentation.viewmodel.SendNotificationViewModel

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SendNotificationScreen(
    onBack:    () -> Unit,
    viewModel: SendNotificationViewModel = hiltViewModel(),
) {
    val state by viewModel.state.collectAsState()
    var subject   by remember { mutableStateOf("") }
    var message   by remember { mutableStateOf("") }
    var userIdStr by remember { mutableStateOf("") }   // vacío = envío masivo

    val snackbarHostState = remember { SnackbarHostState() }

    // Mostrar resultado exitoso
    LaunchedEffect(state.result) {
        state.result?.let { result ->
            val msg = buildString {
                append("✅ Enviado a ${result.sent} usuario(s)")
                if (result.failed > 0) append(" — ${result.failed} fallido(s)")
            }
            snackbarHostState.showSnackbar(msg)
            viewModel.clearResult()
        }
    }

    // Mostrar error
    LaunchedEffect(state.error) {
        state.error?.let {
            snackbarHostState.showSnackbar("Error: $it")
            viewModel.clearError()
        }
    }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Enviar notificación") },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(
                            imageVector        = Icons.AutoMirrored.Filled.ArrowBack,
                            contentDescription = "Volver",
                        )
                    }
                },
            )
        },
        snackbarHost = { SnackbarHost(snackbarHostState) },
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .padding(horizontal = 24.dp)
                .verticalScroll(rememberScrollState()),
        ) {
            Spacer(Modifier.height(16.dp))

            // ── Asunto ────────────────────────────────────────────────────────
            OutlinedTextField(
                value           = subject,
                onValueChange   = { subject = it },
                label           = { Text("Asunto") },
                singleLine      = true,
                enabled         = !state.isLoading,
                keyboardOptions = KeyboardOptions(imeAction = ImeAction.Next),
                modifier        = Modifier.fillMaxWidth(),
            )

            Spacer(Modifier.height(12.dp))

            // ── Mensaje ───────────────────────────────────────────────────────
            OutlinedTextField(
                value         = message,
                onValueChange = { message = it },
                label         = { Text("Mensaje") },
                minLines      = 5,
                maxLines      = 10,
                enabled       = !state.isLoading,
                modifier      = Modifier.fillMaxWidth(),
            )

            Spacer(Modifier.height(12.dp))

            // ── ID de usuario (opcional) ──────────────────────────────────────
            OutlinedTextField(
                value           = userIdStr,
                onValueChange   = { userIdStr = it.filter { c -> c.isDigit() } },
                label           = { Text("ID de usuario (opcional)") },
                placeholder     = { Text("Dejar vacío para envío masivo") },
                singleLine      = true,
                enabled         = !state.isLoading,
                keyboardOptions = KeyboardOptions(
                    keyboardType = KeyboardType.Number,
                    imeAction    = ImeAction.Done,
                ),
                supportingText  = {
                    Text(
                        if (userIdStr.isBlank())
                            "Sin ID → se envía a todos los usuarios activos no-staff"
                        else
                            "Con ID → se envía solo al usuario #$userIdStr"
                    )
                },
                modifier = Modifier.fillMaxWidth(),
            )

            Spacer(Modifier.height(24.dp))

            // ── Botón de envío ────────────────────────────────────────────────
            val userId      = userIdStr.toIntOrNull()
            val isFormValid = subject.isNotBlank() && message.isNotBlank()

            Button(
                onClick  = { viewModel.send(subject.trim(), message.trim(), userId) },
                enabled  = isFormValid && !state.isLoading,
                modifier = Modifier.fillMaxWidth(),
            ) {
                if (state.isLoading) {
                    CircularProgressIndicator(
                        modifier    = Modifier.size(18.dp),
                        color       = MaterialTheme.colorScheme.onPrimary,
                        strokeWidth = 2.dp,
                    )
                    Spacer(Modifier.width(8.dp))
                    Text("Enviando...")
                } else {
                    Icon(
                        imageVector        = Icons.Default.Send,
                        contentDescription = null,
                        modifier           = Modifier.size(18.dp),
                    )
                    Spacer(Modifier.width(8.dp))
                    Text(
                        text = if (userId != null) "Enviar al usuario #$userId"
                               else "Enviar a todos"
                    )
                }
            }

            Spacer(Modifier.height(16.dp))
        }
    }
}
```

> **Botón dinámico:** el texto del botón cambia entre "Enviar al usuario #N" y "Enviar a todos" según si el campo de ID está relleno o vacío, dando feedback visual inmediato sobre el tipo de envío.

---

## 13.14 Actualizar `ProfileScreen` — acceso para staff

Añade `onSendNotification` a la firma y un `ListItem` visible solo cuando `isStaff == true`:

`presentation/ui/client/profile/ProfileScreen.kt  — fragmento relevante`
```kotlin

import androidx.compose.foundation.clickable
import androidx.compose.material.icons.filled.Send
import androidx.compose.material.icons.automirrored.filled.ArrowForward

@Composable
fun ProfileScreen(
    onEditProfile:      () -> Unit = {},
    onLogout:           () -> Unit = {},
    onSendNotification: () -> Unit = {},    // ← nuevo parámetro
    viewModel:          ProfileViewModel = hiltViewModel(),
) {
    // ... código existente ...

    // DENTRO del Column, en el bloque de opciones antes del logout:
    val profile = state.profile

    if (profile?.isStaff == true) {
        HorizontalDivider()

        ListItem(
            headlineContent   = {
                Text("Enviar notificación", fontWeight = FontWeight.Medium)
            },
            supportingContent = {
                Text("Envía un correo a uno o todos los usuarios")
            },
            leadingContent    = {
                Icon(
                    imageVector        = Icons.Default.Send,
                    contentDescription = null,
                    tint               = MaterialTheme.colorScheme.primary,
                )
            },
            trailingContent   = {
                Icon(
                    imageVector        = Icons.AutoMirrored.Filled.ArrowForward,
                    contentDescription = null,
                )
            },
            modifier = Modifier.clickable(onClick = onSendNotification),
        )

        HorizontalDivider()
    }
}
```

> La opción solo se renderiza cuando `profile?.isStaff == true`. Si el usuario no es staff no verá el ítem, y el backend rechazará con `403` cualquier intento directo al endpoint.

---

## 13.15 Actualizar la navegación

Primero añade la ruta nueva a `Screen.kt`:

`presentation/navigation/Screen.kt  (añadir en el bloque Admin o Client)`
```kotlin

data object SendNotification : Screen("send-notification")
```

Luego añade el composable en `NavGraph.kt`:

`presentation/navigation/NavGraph.kt  (añadir dentro del NavHost)`
```kotlin

// ── Notificaciones de staff ───────────────────────────────────────────────────
composable(Screen.SendNotification.route) {
    if (!isStaff) {
        LaunchedEffect(Unit) {
            navController.popBackStack()
        }
        return@composable
    }
    SendNotificationScreen(
        onBack = { navController.popBackStack() },
    )
}
```

Actualiza el composable de `ProfileScreen` en `NavGraph.kt`:

`presentation/navigation/NavGraph.kt  (actualizar composable existente)`
```kotlin

composable(Screen.Profile.route) {
    if (!isAuthenticated) {
        LaunchedEffect(Unit) {
            navController.navigate(Screen.Login.route) { popUpTo(Screen.Home.route) }
        }
    } else {
        ProfileScreen(
            onLogout = {
                authViewModel.logout()
                navController.navigate(Screen.Login.route) {
                    popUpTo(0) { inclusive = true }
                }
            },
            onSendNotification = { navController.navigate(Screen.SendNotification.route) },  // ← nuevo
        )
    }
}
```

---

## 🚦 PARADA 2 — Enviar notificación

> **Qué está listo:** DTOs añadidos a `UserDto.kt`, endpoint en `UserApi`, método en `UserRepository`, `SendNotificationViewModel`, `SendNotificationScreen`, acceso desde `ProfileScreen` para staff, ruta en NavHost.
> **Qué vas a probar:** envío individual, envío masivo y que un usuario normal no puede acceder.

**Prueba 1 — Envío individual**

1. Autenticarse como usuario **staff**.
2. Ir a `ProfileScreen` → debe aparecer el ítem **"Enviar notificación"**.
3. Pulsarlo → navega a `SendNotificationScreen`.
4. Completar:
   - Asunto: `¡Promoción exclusiva!`
   - Mensaje: `Tienes un 20 % de descuento este mes. Usa el código PROMO20.`
   - ID de usuario: `2` (o el ID de un usuario no-staff registrado)
5. El botón muestra **"Enviar al usuario #2"** → pulsarlo.
6. Snackbar: `✅ Enviado a 1 usuario(s)`.
7. Abrir Gmail del usuario #2 → correo con saludo personalizado y el mensaje.

---

**Prueba 2 — Envío masivo**

1. En `SendNotificationScreen` dejar el campo **ID de usuario** vacío.
2. Completar asunto y mensaje.
3. El botón muestra **"Enviar a todos"** → pulsarlo.
4. Snackbar: `✅ Enviado a N usuario(s)` donde N es el número de usuarios activos no-staff con email.

---

**Prueba 3 — Sin permiso de staff**

1. Cerrar sesión. Autenticarse como usuario **normal** (no staff).
2. Ir a `ProfileScreen` → el ítem **"Enviar notificación" no debe aparecer**.
3. Verificación desde Postman con el token del usuario normal:
   ```
   POST /api/emails/send/
   Authorization: Bearer <access_token_usuario_normal>

   { "subject": "test", "message": "test" }
   ```
   Respuesta esperada: `403 Forbidden`.

---

**Casos de borde:**

| Caso | Resultado esperado |
|------|--------------------|
| ID de un usuario staff | Error `400` del backend en Snackbar |
| ID de usuario inactivo | Error `400` del backend en Snackbar |
| Asunto o mensaje vacío | Botón deshabilitado (validación en UI) |
| 0 usuarios no-staff activos con email | `✅ Enviado a 0 usuario(s)` |
| Un SMTP falla en envío masivo | `✅ Enviado a N usuario(s) — 1 fallido(s)` |

---

## ✅ Checkpoint Módulo 13

| # | Verificación | Resultado esperado |
|---|---|---|
| 1 | Pulsar "¿Olvidaste tu contraseña?" en `LoginScreen` | Navega a `ForgotPasswordScreen` |
| 2 | Ingresar email de usuario registrado → "Enviar enlace" | Muestra pantalla de confirmación; correo llega a Gmail |
| 3 | Ingresar email inexistente → "Enviar enlace" | Misma pantalla de confirmación (anti-enumeración) |
| 4 | Pulsar "Tengo el código" | Navega a `ResetPasswordConfirmScreen` |
| 5 | Pegar uid + token del correo + nueva contraseña → enviar | `200 OK`; navega a Login con Snackbar "Contraseña actualizada" |
| 6 | Login con la nueva contraseña | Nuevos tokens JWT, acceso al home |
| 7 | Reuse del mismo token | Error en Snackbar: "Token inválido o expirado" |
| 8 | Contraseñas que no coinciden en el formulario | Botón deshabilitado, texto de error visible |
| 9 | Usuario staff en `ProfileScreen` | Ítem "Enviar notificación" visible |
| 10 | Usuario normal en `ProfileScreen` | Ítem "Enviar notificación" **no** visible |
| 11 | Envío individual con user_id válido | Snackbar `✅ Enviado a 1 usuario(s)`; correo en Gmail del destinatario |
| 12 | Envío masivo sin user_id | Snackbar con el número de usuarios alcanzados |
| 13 | Formulario con asunto o mensaje vacíos | Botón deshabilitado |

---

## Resumen

En este módulo implementamos los dos flujos de email que requieren acción explícita del usuario en la app:

| Componente | Función |
|-----------|---------|
| `PasswordResetRequestDto`, `PasswordResetConfirmDto`, `MessageDto` | DTOs de reset — añadidos a `AuthDto.kt` |
| `SendNotificationDto`, `NotificationResultDto` | DTOs de notificaciones — añadidos a `UserDto.kt` |
| `AuthRepository` + `AuthRepositoryImpl` | Extendidos con `requestReset` y `confirmReset` |
| `UserRepository` + `UserRepositoryImpl` | Extendidos con `sendNotification` |
| `ForgotPasswordViewModel` | Estado de la solicitud de reset (emailSent, isLoading, error) |
| `ResetPasswordConfirmViewModel` | Estado de la confirmación del reset (resetSuccess, isLoading, error) |
| `SendNotificationViewModel` | Estado del envío de notificaciones (result, isLoading, error) |
| `ForgotPasswordScreen` | Ingreso de email y pantalla de confirmación de envío |
| `ResetPasswordConfirmScreen` | Ingreso de uid + token + nueva contraseña |
| `SendNotificationScreen` | Formulario de envío individual o masivo (solo staff) |

Los correos de bienvenida y confirmación de orden no necesitan pantallas adicionales: el backend los dispara automáticamente mediante señales Django al crear un usuario o una orden desde cualquier cliente.

> **Mejora futura — Deep links:** para eliminar la copia manual del uid y token se puede configurar un *Android App Link* o un *deep link* personalizado (`shopapp://password-reset/confirm?uid=...&token=...`). Esto requiere actualizar `FRONTEND_URL` en el backend, añadir un Intent Filter en `AndroidManifest.xml` y modificar `ResetPasswordConfirmScreen` para leer los argumentos de navegación en lugar de campos de texto.
