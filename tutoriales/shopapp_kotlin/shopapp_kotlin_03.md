# ShopApp Android — Módulo 3
## Auth — Login, Registro, AuthViewModel y persistencia de sesión

> **Objetivo:** Sistema de autenticación completo: repositorio de auth, ViewModel con estado reactivo, pantallas de Login y Registro, y restauración de sesión al arrancar la app.
> **Checkpoint final:** login con credenciales reales, sesión persistida al cerrar y reabrir la app, redirección automática según el rol.

---

## 3.1 Repositorio de auth — `domain/repository/AuthRepository.kt`

```kotlin
// domain/repository/AuthRepository.kt
package com.shopapp.domain.repository

import com.shopapp.data.local.TokenDataStore
import com.shopapp.domain.model.LoggedUser

interface AuthRepository {
    suspend fun login(username: String, password: String): Result<LoggedUser>
    suspend fun register(
        username: String,
        email: String,
        password: String,
        password2: String,
    ): Result<LoggedUser>
    suspend fun logout(): Result<Unit>
    suspend fun getStoredUser(): TokenDataStore.UserSnapshot?
    suspend fun isLoggedIn(): Boolean
}
```

### `data/repository/AuthRepositoryImpl.kt`

```kotlin
// data/repository/AuthRepositoryImpl.kt
package com.shopapp.data.repository

import com.shopapp.data.local.TokenDataStore
import com.shopapp.data.remote.api.AuthApi
import com.shopapp.data.remote.dto.*
import com.shopapp.domain.model.LoggedUser
import com.shopapp.domain.repository.AuthRepository
import kotlinx.coroutines.flow.first
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class AuthRepositoryImpl @Inject constructor(
    private val api:            AuthApi,
    private val tokenDataStore: TokenDataStore,
) : AuthRepository {

    override suspend fun login(username: String, password: String): Result<LoggedUser> =
        runCatching {
            val response = api.login(LoginRequest(username, password))
            if (!response.isSuccessful) {
                val errorBody = response.errorBody()?.string() ?: ""
                error(parseErrorMessage(errorBody, response.code()))
            }
            val body = response.body()!!
            tokenDataStore.saveTokens(body.access, body.refresh)
            tokenDataStore.saveUser(body.userId, body.username, body.email, body.isStaff)
            LoggedUser(body.userId, body.username, body.email, body.isStaff)
        }

    override suspend fun register(
        username: String,
        email: String,
        password: String,
        password2: String,
    ): Result<LoggedUser> = runCatching {
        val response = api.register(RegisterRequest(username, email, password, password2))
        if (!response.isSuccessful) {
            val errorBody = response.errorBody()?.string() ?: ""
            error(parseErrorMessage(errorBody, response.code()))
        }
        val body = response.body()!!
        tokenDataStore.saveTokens(body.access, body.refresh)
        tokenDataStore.saveUser(body.userId, body.username, body.email, body.isStaff)
        LoggedUser(body.userId, body.username, body.email, body.isStaff)
    }

    override suspend fun logout(): Result<Unit> = runCatching {
        val refresh = tokenDataStore.getRefreshToken()
        if (refresh != null) {
            runCatching { api.logout(LogoutRequest(refresh)) }
        }
        tokenDataStore.clearSession()
    }

    override suspend fun getStoredUser(): TokenDataStore.UserSnapshot? =
        tokenDataStore.userSnapshot.first()

    override suspend fun isLoggedIn(): Boolean =
        !tokenDataStore.getAccessToken().isNullOrBlank()

    // Extrae el mensaje de error legible del JSON de Django
    private fun parseErrorMessage(body: String, code: Int): String {
        return try {
            val map = com.google.gson.Gson()
                .fromJson(body, Map::class.java)
            map["detail"]?.toString()
                ?: map["non_field_errors"]?.toString()
                ?: map.values.firstOrNull()?.toString()
                ?: "Error $code"
        } catch (e: Exception) {
            "Error $code"
        }
    }
}
```

---

## 3.2 Actualizar `di/RepositoryModule.kt`

```kotlin
// di/RepositoryModule.kt
package com.shopapp.di

import com.shopapp.data.repository.AuthRepositoryImpl
import com.shopapp.data.repository.CategoryRepositoryImpl
import com.shopapp.domain.repository.AuthRepository
import com.shopapp.domain.repository.CategoryRepository
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds @Singleton
    abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository

    @Binds @Singleton
    abstract fun bindCategoryRepository(impl: CategoryRepositoryImpl): CategoryRepository
}
```

---

## 3.3 Estado de UI para Auth

```kotlin
// presentation/ui/auth/AuthUiState.kt
package com.shopapp.presentation.ui.auth

import com.shopapp.domain.model.LoggedUser

sealed interface AuthUiState {
    data object Idle        : AuthUiState
    data object Loading     : AuthUiState
    data class  Success(val user: LoggedUser) : AuthUiState
    data class  Error(val message: String)    : AuthUiState
}
```

---

## 3.4 AuthViewModel — `presentation/viewmodel/AuthViewModel.kt`

```kotlin
// presentation/viewmodel/AuthViewModel.kt
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.data.local.TokenDataStore
import com.shopapp.domain.model.LoggedUser
import com.shopapp.domain.repository.AuthRepository
import com.shopapp.presentation.ui.auth.AuthUiState
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class AuthViewModel @Inject constructor(
    private val authRepository: AuthRepository,
    private val tokenDataStore: TokenDataStore,
) : ViewModel() {

    // ── Estado de la UI ───────────────────────────────────────
    private val _uiState = MutableStateFlow<AuthUiState>(AuthUiState.Idle)
    val uiState: StateFlow<AuthUiState> = _uiState.asStateFlow()

    // ── Usuario logueado (Flow reactivo) ──────────────────────
    private val _currentUser = MutableStateFlow<LoggedUser?>(null)
    val currentUser: StateFlow<LoggedUser?> = _currentUser.asStateFlow()

    val isAuthenticated: StateFlow<Boolean> = _currentUser
        .map { it != null }
        .stateIn(viewModelScope, SharingStarted.Eagerly, false)

    val isStaff: StateFlow<Boolean> = _currentUser
        .map { it?.isStaff == true }
        .stateIn(viewModelScope, SharingStarted.Eagerly, false)

    // ── Estado de carga inicial ───────────────────────────────
    private val _isCheckingSession = MutableStateFlow(true)
    val isCheckingSession: StateFlow<Boolean> = _isCheckingSession.asStateFlow()

    init {
        restoreSession()
    }

    // Restaurar sesión desde DataStore al arrancar la app
    private fun restoreSession() {
        viewModelScope.launch {
            try {
                val snapshot = authRepository.getStoredUser()
                if (snapshot != null && authRepository.isLoggedIn()) {
                    _currentUser.value = LoggedUser(
                        id       = snapshot.id,
                        username = snapshot.username,
                        email    = snapshot.email,
                        isStaff  = snapshot.isStaff,
                    )
                }
            } finally {
                _isCheckingSession.value = false
            }
        }
    }

    // ── Login ─────────────────────────────────────────────────
    fun login(username: String, password: String) {
        if (_uiState.value is AuthUiState.Loading) return
        viewModelScope.launch {
            _uiState.value = AuthUiState.Loading
            authRepository.login(username.trim(), password)
                .onSuccess { user ->
                    _currentUser.value = user
                    _uiState.value     = AuthUiState.Success(user)
                }
                .onFailure { e ->
                    _uiState.value = AuthUiState.Error(e.message ?: "Error al iniciar sesión")
                }
        }
    }

    // ── Registro ──────────────────────────────────────────────
    fun register(username: String, email: String, password: String, password2: String) {
        if (_uiState.value is AuthUiState.Loading) return
        viewModelScope.launch {
            _uiState.value = AuthUiState.Loading
            authRepository.register(username.trim(), email.trim(), password, password2)
                .onSuccess { user ->
                    _currentUser.value = user
                    _uiState.value     = AuthUiState.Success(user)
                }
                .onFailure { e ->
                    _uiState.value = AuthUiState.Error(e.message ?: "Error al registrarse")
                }
        }
    }

    // ── Logout ────────────────────────────────────────────────
    fun logout() {
        viewModelScope.launch {
            authRepository.logout()
            _currentUser.value = null
            _uiState.value     = AuthUiState.Idle
        }
    }

    fun clearError() {
        if (_uiState.value is AuthUiState.Error) {
            _uiState.value = AuthUiState.Idle
        }
    }
}
```

---

## 3.5 Componentes UI reutilizables

### `presentation/components/ShopTextField.kt`

```kotlin
// presentation/components/ShopTextField.kt
package com.shopapp.presentation.components

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.text.KeyboardOptions
import androidx.compose.material.icons.filled.Visibility
import androidx.compose.material.icons.filled.VisibilityOff
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.input.*
import com.shopapp.theme.*

@Composable
fun ShopTextField(
    value:         String,
    onValueChange: (String) -> Unit,
    label:         String,
    modifier:      Modifier = Modifier,
    placeholder:   String   = "",
    isError:       Boolean  = false,
    errorMessage:  String?  = null,
    isPassword:    Boolean  = false,
    trailingIcon:  @Composable (() -> Unit)? = null,
    keyboardType:  KeyboardType = KeyboardType.Text,
    imeAction:     ImeAction    = ImeAction.Next,
    enabled:       Boolean      = true,
) {
    var passwordVisible by remember { mutableStateOf(false) }

    val visualTransformation = when {
        isPassword && !passwordVisible -> PasswordVisualTransformation()
        else                           -> VisualTransformation.None
    }

    Column(modifier = modifier) {
        OutlinedTextField(
            value           = value,
            onValueChange   = onValueChange,
            label           = { Text(label) },
            placeholder     = { Text(placeholder, color = TextFaint) },
            isError         = isError,
            visualTransformation = visualTransformation,
            keyboardOptions = KeyboardOptions(
                keyboardType = keyboardType,
                imeAction = imeAction,
            ),
            enabled         = enabled,
            singleLine      = true,
            modifier        = Modifier.fillMaxWidth(),
            colors          = OutlinedTextFieldDefaults.colors(
                focusedBorderColor   = Accent,
                focusedLabelColor    = Accent,
                cursorColor          = Accent,
                unfocusedBorderColor = Border,
                unfocusedLabelColor  = TextSecondary,
                errorBorderColor     = Error,
                errorLabelColor      = Error,
            ),
            trailingIcon = if (isPassword) {
                {
                    IconButton(onClick = { passwordVisible = !passwordVisible }) {
                        Icon(
                            imageVector = if (passwordVisible)
                                androidx.compose.material.icons.Icons.Default.VisibilityOff
                            else
                                androidx.compose.material.icons.Icons.Default.Visibility,
                            contentDescription = if (passwordVisible) "Ocultar" else "Mostrar",
                            tint = TextSecondary,
                        )
                    }
                }
            } else trailingIcon,
        )
        if (isError && errorMessage != null) {
            Text(
                text  = errorMessage,
                color = Error,
                style = MaterialTheme.typography.bodySmall,
            )
        }
    }
}

```

### `presentation/components/ShopButton.kt`

```kotlin
// presentation/components/ShopButton.kt
package com.shopapp.presentation.components

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.shopapp.theme.Accent
import com.shopapp.theme.AccentOnDark

@Composable
fun ShopButton(
    text:      String,
    onClick:   () -> Unit,
    modifier:  Modifier = Modifier,
    isLoading: Boolean  = false,
    enabled:   Boolean  = true,
) {
    Button(
        onClick  = onClick,
        enabled  = enabled && !isLoading,
        modifier = modifier.fillMaxWidth().height(52.dp),
        colors   = ButtonDefaults.buttonColors(
            containerColor         = Accent,
            contentColor           = AccentOnDark,
            disabledContainerColor = Accent.copy(alpha = 0.5f),
            disabledContentColor   = AccentOnDark.copy(alpha = 0.5f),
        ),
        shape = MaterialTheme.shapes.medium,
    ) {
        if (isLoading) {
            CircularProgressIndicator(
                color     = AccentOnDark,
                modifier  = Modifier.size(20.dp),
                strokeWidth = 2.dp,
            )
            Spacer(Modifier.width(8.dp))
        }
        Text(text = if (isLoading) "Cargando..." else text, style = MaterialTheme.typography.labelLarge)
    }
}

@Composable
fun ShopOutlinedButton(
    text:     String,
    onClick:  () -> Unit,
    modifier: Modifier = Modifier,
    enabled:  Boolean  = true,
) {
    OutlinedButton(
        onClick  = onClick,
        enabled  = enabled,
        modifier = modifier.fillMaxWidth().height(52.dp),
        colors   = ButtonDefaults.outlinedButtonColors(contentColor = Accent),
        border   = ButtonDefaults.outlinedButtonBorder.copy(
            brush = androidx.compose.ui.graphics.SolidColor(Accent)
        ),
        shape    = MaterialTheme.shapes.medium,
    ) {
        Text(text = text, style = MaterialTheme.typography.labelLarge)
    }
}
```

### `presentation/components/LoadingScreen.kt`

```kotlin
// presentation/components/LoadingScreen.kt
package com.shopapp.presentation.components

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import com.shopapp.theme.*

@Composable
fun LoadingScreen(message: String = "Cargando...") {
    Box(
        modifier          = Modifier.fillMaxSize().background(Background),
        contentAlignment  = Alignment.Center,
    ) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            CircularProgressIndicator(color = Accent, strokeWidth = 3.dp)
            Spacer(modifier = Modifier.height(16.dp))
            Text(
                text      = message,
                color     = TextSecondary,
                style     = MaterialTheme.typography.bodyMedium,
            )
        }
    }
}

@Composable
fun ErrorScreen(message: String, onRetry: (() -> Unit)? = null) {
    Box(
        modifier          = Modifier.fillMaxSize().background(Background),
        contentAlignment  = Alignment.Center,
    ) {
        Column(
            horizontalAlignment = Alignment.CenterHorizontally,
            modifier            = Modifier.padding(24.dp),
        ) {
            Text("⚠️", style = MaterialTheme.typography.displayMedium)
            Spacer(Modifier.height(12.dp))
            Text(
                text      = "Algo salió mal",
                color     = TextPrimary,
                style     = MaterialTheme.typography.titleMedium,
                fontWeight = FontWeight.Bold,
            )
            Spacer(Modifier.height(8.dp))
            Text(
                text  = message,
                color = TextSecondary,
                style = MaterialTheme.typography.bodyMedium,
            )
            if (onRetry != null) {
                Spacer(Modifier.height(24.dp))
                ShopButton(text = "Reintentar", onClick = onRetry, modifier = Modifier.width(200.dp))
            }
        }
    }
}
```

---

## 3.6 Pantalla Login — `presentation/ui/auth/LoginScreen.kt`

```kotlin
// presentation/ui/auth/LoginScreen.kt
package com.shopapp.presentation.ui.auth

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.presentation.components.*
import com.shopapp.presentation.viewmodel.AuthViewModel
import com.shopapp.theme.*

@Composable
fun LoginScreen(
    onLoginSuccess:  (isStaff: Boolean) -> Unit,
    onNavigateToRegister: () -> Unit,
    viewModel: AuthViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsState()

    var username by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }

    // Navegar cuando el login es exitoso
    LaunchedEffect(uiState) {
        if (uiState is AuthUiState.Success) {
            val user = (uiState as AuthUiState.Success).user
            onLoginSuccess(user.isStaff)
        }
    }

    val isLoading = uiState is AuthUiState.Loading
    val errorMsg  = (uiState as? AuthUiState.Error)?.message

    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(Background),
    ) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .verticalScroll(rememberScrollState())
                .padding(horizontal = 24.dp)
                .padding(top = 80.dp, bottom = 40.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
        ) {
            // Logo
            Text(
                text       = "ShopApp",
                fontSize   = 36.sp,
                fontWeight = FontWeight.Bold,
                color      = Accent,
            )
            Spacer(Modifier.height(8.dp))
            Text(
                text  = "Inicia sesión en tu cuenta",
                color = TextSecondary,
                style = MaterialTheme.typography.bodyMedium,
            )
            Spacer(Modifier.height(40.dp))

            // Card del formulario
            Surface(
                shape            = MaterialTheme.shapes.large,
                color            = Surface,
                tonalElevation   = 0.dp,
                modifier         = Modifier.fillMaxWidth(),
            ) {
                Column(modifier = Modifier.padding(24.dp)) {

                    // Error general
                    if (errorMsg != null) {
                        Surface(
                            color  = Error.copy(alpha = 0.1f),
                            shape  = MaterialTheme.shapes.small,
                            modifier = Modifier.fillMaxWidth(),
                        ) {
                            Text(
                                text     = errorMsg,
                                color    = Error,
                                style    = MaterialTheme.typography.bodySmall,
                                modifier = Modifier.padding(12.dp),
                            )
                        }
                        Spacer(Modifier.height(16.dp))
                    }

                    // Campo usuario
                    ShopTextField(
                        value         = username,
                        onValueChange = { username = it; viewModel.clearError() },
                        label         = "Usuario",
                        placeholder   = "tu_usuario",
                        enabled       = !isLoading,
                        imeAction     = ImeAction.Next,
                    )
                    Spacer(Modifier.height(16.dp))

                    // Campo contraseña
                    ShopTextField(
                        value         = password,
                        onValueChange = { password = it; viewModel.clearError() },
                        label         = "Contraseña",
                        placeholder   = "••••••••",
                        isPassword    = true,
                        enabled       = !isLoading,
                        keyboardType  = KeyboardType.Password,
                        imeAction     = ImeAction.Done,
                    )
                    Spacer(Modifier.height(24.dp))

                    // Botón
                    ShopButton(
                        text      = "Iniciar sesión",
                        onClick   = { viewModel.login(username, password) },
                        isLoading = isLoading,
                        enabled   = username.isNotBlank() && password.isNotBlank(),
                    )
                }
            }

            Spacer(Modifier.height(24.dp))

            // Link a registro
            Row(verticalAlignment = Alignment.CenterVertically) {
                Text(
                    text  = "¿No tienes cuenta? ",
                    color = TextSecondary,
                    style = MaterialTheme.typography.bodyMedium,
                )
                TextButton(onClick = onNavigateToRegister) {
                    Text(
                        text  = "Regístrate",
                        color = Accent,
                        style = MaterialTheme.typography.bodyMedium,
                        fontWeight = FontWeight.SemiBold,
                    )
                }
            }
        }
    }
}
```

---

## 3.7 Pantalla Registro — `presentation/ui/auth/RegisterScreen.kt`

```kotlin
// presentation/ui/auth/RegisterScreen.kt
package com.shopapp.presentation.ui.auth

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.ImeAction
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.presentation.components.*
import com.shopapp.presentation.viewmodel.AuthViewModel
import com.shopapp.theme.*

@Composable
fun RegisterScreen(
    onRegisterSuccess:  (isStaff: Boolean) -> Unit,
    onNavigateToLogin:  () -> Unit,
    viewModel: AuthViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsState()

    var username  by remember { mutableStateOf("") }
    var email     by remember { mutableStateOf("") }
    var password  by remember { mutableStateOf("") }
    var password2 by remember { mutableStateOf("") }

    // Validación local
    val passwordMismatch = password.isNotEmpty() &&
                           password2.isNotEmpty() &&
                           password != password2

    LaunchedEffect(uiState) {
        if (uiState is AuthUiState.Success) {
            val user = (uiState as AuthUiState.Success).user
            onRegisterSuccess(user.isStaff)
        }
    }

    val isLoading = uiState is AuthUiState.Loading
    val errorMsg  = (uiState as? AuthUiState.Error)?.message

    Box(
        modifier = Modifier
            .fillMaxSize()
            .background(Background),
    ) {
        Column(
            modifier = Modifier
                .fillMaxSize()
                .verticalScroll(rememberScrollState())
                .padding(horizontal = 24.dp)
                .padding(top = 60.dp, bottom = 40.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
        ) {
            Text(
                text       = "ShopApp",
                fontSize   = 36.sp,
                fontWeight = FontWeight.Bold,
                color      = Accent,
            )
            Spacer(Modifier.height(8.dp))
            Text(
                text  = "Crea tu cuenta gratis",
                color = TextSecondary,
                style = MaterialTheme.typography.bodyMedium,
            )
            Spacer(Modifier.height(32.dp))

            Surface(
                shape          = MaterialTheme.shapes.large,
                color          = Surface,
                tonalElevation = 0.dp,
                modifier       = Modifier.fillMaxWidth(),
            ) {
                Column(modifier = Modifier.padding(24.dp)) {

                    if (errorMsg != null) {
                        Surface(
                            color  = Error.copy(alpha = 0.1f),
                            shape  = MaterialTheme.shapes.small,
                            modifier = Modifier.fillMaxWidth(),
                        ) {
                            Text(
                                text     = errorMsg,
                                color    = Error,
                                style    = MaterialTheme.typography.bodySmall,
                                modifier = Modifier.padding(12.dp),
                            )
                        }
                        Spacer(Modifier.height(16.dp))
                    }

                    ShopTextField(
                        value         = username,
                        onValueChange = { username = it; viewModel.clearError() },
                        label         = "Usuario",
                        placeholder   = "mínimo 3 caracteres",
                        enabled       = !isLoading,
                        isError       = username.isNotEmpty() && username.length < 3,
                        errorMessage  = "Mínimo 3 caracteres",
                        imeAction     = ImeAction.Next,
                    )
                    Spacer(Modifier.height(14.dp))

                    ShopTextField(
                        value         = email,
                        onValueChange = { email = it; viewModel.clearError() },
                        label         = "Email",
                        placeholder   = "tu@email.com",
                        enabled       = !isLoading,
                        keyboardType  = KeyboardType.Email,
                        isError       = email.isNotEmpty() && !email.contains("@"),
                        errorMessage  = "Email inválido",
                        imeAction     = ImeAction.Next,
                    )
                    Spacer(Modifier.height(14.dp))

                    ShopTextField(
                        value         = password,
                        onValueChange = { password = it; viewModel.clearError() },
                        label         = "Contraseña",
                        placeholder   = "mínimo 8 caracteres",
                        isPassword    = true,
                        enabled       = !isLoading,
                        keyboardType  = KeyboardType.Password,
                        isError       = password.isNotEmpty() && password.length < 8,
                        errorMessage  = "Mínimo 8 caracteres",
                        imeAction     = ImeAction.Next,
                    )
                    Spacer(Modifier.height(14.dp))

                    ShopTextField(
                        value         = password2,
                        onValueChange = { password2 = it; viewModel.clearError() },
                        label         = "Confirmar contraseña",
                        placeholder   = "repite la contraseña",
                        isPassword    = true,
                        enabled       = !isLoading,
                        keyboardType  = KeyboardType.Password,
                        isError       = passwordMismatch,
                        errorMessage  = "Las contraseñas no coinciden",
                        imeAction     = ImeAction.Done,
                    )
                    Spacer(Modifier.height(24.dp))

                    val canSubmit = username.length >= 3 &&
                                   email.contains("@") &&
                                   password.length >= 8 &&
                                   !passwordMismatch &&
                                   !isLoading

                    ShopButton(
                        text      = "Crear mi cuenta",
                        onClick   = { viewModel.register(username, email, password, password2) },
                        isLoading = isLoading,
                        enabled   = canSubmit,
                    )
                }
            }

            Spacer(Modifier.height(24.dp))

            Row(verticalAlignment = Alignment.CenterVertically) {
                Text(
                    text  = "¿Ya tienes cuenta? ",
                    color = TextSecondary,
                    style = MaterialTheme.typography.bodyMedium,
                )
                TextButton(onClick = onNavigateToLogin) {
                    Text(
                        text       = "Inicia sesión",
                        color      = Accent,
                        style      = MaterialTheme.typography.bodyMedium,
                        fontWeight = FontWeight.SemiBold,
                    )
                }
            }
        }
    }
}
```

---

## 3.8 NavGraph provisional — `presentation/navigation/Screen.kt`

```kotlin
// presentation/navigation/Screen.kt
package com.shopapp.presentation.navigation

sealed class Screen(val route: String) {
    // Auth
    data object Login    : Screen("login")
    data object Register : Screen("register")

    // Public
    data object Home     : Screen("home")
    data object Catalog  : Screen("catalog")
    data class  Product(val id: Int = 0) : Screen("product/{id}") {
        fun createRoute(id: Int) = "product/$id"
    }
    data object Cart     : Screen("cart")

    // Client
    data object Orders       : Screen("orders")
    data class  OrderDetail(val id: Int = 0) : Screen("orders/{id}") {
        fun createRoute(id: Int) = "orders/$id"
    }
    data object Profile : Screen("profile")

    // Admin
    data object AdminDashboard  : Screen("admin")
    data object AdminCategories : Screen("admin/categories")
    data object AdminProducts   : Screen("admin/products")
    data object AdminOrders     : Screen("admin/orders")
    data object AdminUsers      : Screen("admin/users")
}
```

---

## 3.9 Actualizar `MainActivity.kt`

```kotlin
package com.shopapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.navigation.compose.*
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.navigation.Screen
import com.shopapp.presentation.ui.auth.LoginScreen
import com.shopapp.presentation.ui.auth.RegisterScreen
import com.shopapp.presentation.viewmodel.AuthViewModel
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
                    ShopApp()
                }
            }
        }
    }
}

@Composable
fun ShopApp() {
    val authViewModel: AuthViewModel = hiltViewModel()

    val isCheckingSession by authViewModel.isCheckingSession.collectAsState()
    val isAuthenticated   by authViewModel.isAuthenticated.collectAsState()
    val isStaff           by authViewModel.isStaff.collectAsState()

    val navController = rememberNavController()

    // Splash mientras valida sesión
    if (isCheckingSession) {
        LoadingScreen(message = "Iniciando ShopApp...")
        return
    }

    val startDestination = when {
        !isAuthenticated -> Screen.Login.route
        isStaff          -> Screen.AdminDashboard.route
        else             -> Screen.Home.route
    }

    NavHost(
        navController = navController,
        startDestination = startDestination,
    ) {

        // ── LOGIN ───────────────────────────────
        composable(Screen.Login.route) {
            LoginScreen(
                onLoginSuccess = { staff ->
                    val dest = if (staff) {
                        Screen.AdminDashboard.route
                    } else {
                        Screen.Home.route
                    }

                    navController.navigate(dest) {
                        popUpTo(Screen.Login.route) { inclusive = true }
                    }
                },
                onNavigateToRegister = {
                    navController.navigate(Screen.Register.route)
                },
            )
        }

        // ── REGISTER ────────────────────────────
        composable(Screen.Register.route) {
            RegisterScreen(
                onRegisterSuccess = { staff ->
                    val dest = if (staff) {
                        Screen.AdminDashboard.route
                    } else {
                        Screen.Home.route
                    }

                    navController.navigate(dest) {
                        popUpTo(Screen.Login.route) { inclusive = true }
                    }
                },
                onNavigateToLogin = {
                    navController.popBackStack()
                },
            )
        }

        // ── HOME (con logout) ───────────────────
        composable(Screen.Home.route) {
            HomeTestScreen(
                onLogout = {
                    authViewModel.logout()
                    navController.navigate(Screen.Login.route) {
                        popUpTo(0) { inclusive = true }
                    }
                }
            )
        }

        // ── ADMIN DASHBOARD (con logout) ────────
        composable(Screen.AdminDashboard.route) {
            AdminDashboardTestScreen(
                onLogout = {
                    authViewModel.logout()
                    navController.navigate(Screen.Login.route) {
                        popUpTo(0) { inclusive = true }
                    }
                }
            )
        }
    }
}

// ─────────────────────────────────────────────
// Home temporal con logout
// ─────────────────────────────────────────────
@Composable
fun HomeTestScreen(
    onLogout: () -> Unit,
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        Text(
            text = "Home — M4",
            style = MaterialTheme.typography.headlineSmall,
        )

        Spacer(modifier = Modifier.height(24.dp))

        Button(onClick = onLogout) {
            Text("Cerrar sesión")
        }
    }
}

// ─────────────────────────────────────────────
// Admin temporal con logout
// ─────────────────────────────────────────────
@Composable
fun AdminDashboardTestScreen(
    onLogout: () -> Unit,
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {
        Text(
            text = "Admin Dashboard — M8",
            style = MaterialTheme.typography.headlineSmall,
        )

        Spacer(modifier = Modifier.height(24.dp))

        Button(onClick = onLogout) {
            Text("Cerrar sesión")
        }
    }
}

```

---

## ✅ Checkpoint Módulo 3

### Crear usuarios de prueba en Django

```bash
uv run python manage.py createsuperuser
# username: admin | password: Admin1234!

uv run python manage.py shell -c "
from django.contrib.auth.models import User
User.objects.create_user('cliente', 'cliente@test.com', 'Pass1234!')
print('Usuario cliente creado')
"
```

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Arrancar la app | Splash "Iniciando ShopApp..." brevemente |
| 2 | Sin sesión guardada | Navega a LoginScreen |
| 3 | Login con credenciales incorrectas | Banner rojo con el mensaje de Django |
| 4 | Login con campo vacío | Botón deshabilitado (no se puede pulsar) |
| 5 | Login con cuenta cliente | Navega a Home (placeholder M4) |
| 6 | Cerrar y reabrir la app | Sesión restaurada, va directamente a Home |
| 7 | Login con cuenta admin (is_staff) | Navega a Admin Dashboard (placeholder M8) |
| 8 | Registro con contraseñas distintas | Error en rojo bajo el campo |
| 9 | Registro válido | Navega a Home |
| 10 | Logcat filtrar "AuthRepo" | Ver logs de login/register |

---

## Resumen

| Elemento | Estado |
|---|---|
| `AuthRepositoryImpl` con parseErrorMessage | ✅ |
| `AuthViewModel` con StateFlow reactivo | ✅ |
| Restauración de sesión en `init {}` | ✅ |
| `AuthUiState` sealed interface | ✅ |
| `LoginScreen` con validación y error inline | ✅ |
| `RegisterScreen` con validación en tiempo real | ✅ |
| `ShopTextField` con toggle de contraseña | ✅ |
| `ShopButton` con estado loading | ✅ |
| `LoadingScreen` y `ErrorScreen` | ✅ |
| `Screen` sealed class para navegación | ✅ |
| NavHost con destino inicial según sesión | ✅ |

**Siguiente módulo →** M4: Navegación completa — NavGraph con guards, tienda pública, Home y Catálogo