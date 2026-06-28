# ShopApp Android — Módulo 11
## Admin CRUD Usuarios — Lista, crear, editar, toggle staff y activar/desactivar

> **Objetivo:** Gestión completa de usuarios desde el panel admin: búsqueda con debounce, filtros por rol/estado, formulario con contraseña opcional, toggle de staff y activar/desactivar optimistas.
> **Checkpoint final:** crear un nuevo usuario staff, verificar que puede acceder al panel admin, y gestionar el rol de usuarios existentes.

---

## 11.1 ViewModel de usuarios admin — `presentation/viewmodel/UsersAdminViewModel.kt`

```kotlin
// presentation/viewmodel/UsersAdminViewModel.kt
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.User
import com.shopapp.domain.model.UserPayload
import com.shopapp.domain.repository.UserRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import javax.inject.Inject

enum class UserRoleFilter(val label: String) {
    ALL("Todos"),
    CLIENTS("Clientes"),
    STAFF("Staff"),
    ACTIVE("Activos"),
    INACTIVE("Inactivos"),
}

data class UsersAdminUiState(
    val users:      List<User>     = emptyList(),
    val isLoading:  Boolean        = false,
    val error:      String?        = null,
    val total:      Int            = 0,
    val search:     String         = "",
    val roleFilter: UserRoleFilter = UserRoleFilter.ALL,
)

sealed interface UserFormState {
    data object Idle                       : UserFormState
    data object Saving                     : UserFormState
    data class  Success(val msg: String)   : UserFormState
    data class  Error(val message: String) : UserFormState
}

@HiltViewModel
class UsersAdminViewModel @Inject constructor(
    private val repository: UserRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(UsersAdminUiState())
    val state: StateFlow<UsersAdminUiState> = _state.asStateFlow()

    private val _formState = MutableStateFlow<UserFormState>(UserFormState.Idle)
    val formState: StateFlow<UserFormState> = _formState.asStateFlow()

    // Filtrado local combinado
    val filtered: StateFlow<List<User>> = _state
        .map { s ->
            s.users
                .filter { u ->
                    s.search.isBlank() ||
                    u.username.contains(s.search, ignoreCase = true) ||
                    u.email.contains(s.search, ignoreCase = true)
                }
                .filter { u ->
                    when (s.roleFilter) {
                        UserRoleFilter.ALL      -> true
                        UserRoleFilter.CLIENTS  -> !u.isStaff
                        UserRoleFilter.STAFF    -> u.isStaff
                        UserRoleFilter.ACTIVE   -> u.isActive
                        UserRoleFilter.INACTIVE -> !u.isActive
                    }
                }
        }
        .stateIn(viewModelScope, SharingStarted.Eagerly, emptyList())

    private var searchJob: Job? = null

    init { load() }

    fun load() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            repository.getUsers()
                .onSuccess { (users, total) ->
                    _state.update { it.copy(users = users, total = total, isLoading = false) }
                }
                .onFailure { e ->
                    _state.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    fun setSearch(query: String) {
        _state.update { it.copy(search = query) }
        // Debounce para búsqueda local (ya es instantánea, pero útil si se cambia a API)
        searchJob?.cancel()
        searchJob = viewModelScope.launch {
            delay(300)
            // Búsqueda ya aplicada por el filtered StateFlow
        }
    }

    fun setRoleFilter(filter: UserRoleFilter) {
        _state.update { it.copy(roleFilter = filter) }
    }

    // Toggle staff — optimista
    fun toggleStaff(id: Int, isStaff: Boolean) {
        _state.update { s ->
            s.copy(users = s.users.map { u ->
                if (u.id == id) u.copy(isStaff = isStaff) else u
            })
        }
        viewModelScope.launch {
            val user = _state.value.users.first { it.id == id }
            repository.updateUser(id, UserPayload(
                username  = user.username,
                email     = user.email,
                firstName = user.firstName,
                lastName  = user.lastName,
                isStaff   = isStaff,
                isActive  = user.isActive,
            )).onFailure {
                // Revertir
                _state.update { s ->
                    s.copy(users = s.users.map { u ->
                        if (u.id == id) u.copy(isStaff = !isStaff) else u
                    })
                }
            }
        }
    }

    // Toggle activo — optimista
    fun toggleActive(id: Int) {
        val user = _state.value.users.find { it.id == id } ?: return
        val next = !user.isActive
        _state.update { s ->
            s.copy(users = s.users.map { u ->
                if (u.id == id) u.copy(isActive = next) else u
            })
        }
        viewModelScope.launch {
            repository.toggleActive(id)
                .onSuccess { serverActive ->
                    _state.update { s ->
                        s.copy(users = s.users.map { u ->
                            if (u.id == id) u.copy(isActive = serverActive) else u
                        })
                    }
                }
                .onFailure {
                    // Revertir
                    _state.update { s ->
                        s.copy(users = s.users.map { u ->
                            if (u.id == id) u.copy(isActive = !next) else u
                        })
                    }
                }
        }
    }

    fun createUser(payload: UserPayload) {
        _formState.value = UserFormState.Saving
        viewModelScope.launch {
            repository.createUser(payload)
                .onSuccess { created ->
                    _state.update { s ->
                        s.copy(users = listOf(created) + s.users, total = s.total + 1)
                    }
                    _formState.value = UserFormState.Success("Usuario creado")
                }
                .onFailure { e ->
                    _formState.value = UserFormState.Error(e.message ?: "Error al crear")
                }
        }
    }

    fun updateUser(id: Int, payload: UserPayload) {
        _formState.value = UserFormState.Saving
        viewModelScope.launch {
            repository.updateUser(id, payload)
                .onSuccess { updated ->
                    _state.update { s ->
                        s.copy(users = s.users.map { if (it.id == id) updated else it })
                    }
                    _formState.value = UserFormState.Success("Usuario actualizado")
                }
                .onFailure { e ->
                    _formState.value = UserFormState.Error(e.message ?: "Error al actualizar")
                }
        }
    }

    fun deleteUser(id: Int) {
        viewModelScope.launch {
            repository.deleteUser(id)
                .onSuccess {
                    _state.update { s ->
                        s.copy(users = s.users.filter { it.id != id }, total = s.total - 1)
                    }
                }
                .onFailure { e ->
                    _state.update { it.copy(error = e.message) }
                }
        }
    }

    fun resetFormState() { _formState.value = UserFormState.Idle }
}
```

---

## 11.2 Formulario de usuario — `presentation/ui/admin/users/UserFormSheet.kt`

```kotlin
// presentation/ui/admin/users/UserFormSheet.kt
package com.shopapp.presentation.ui.admin.users

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.unit.dp
import com.shopapp.domain.model.User
import com.shopapp.domain.model.UserPayload
import com.shopapp.presentation.components.ShopTextField
import com.shopapp.presentation.viewmodel.UserFormState
import com.shopapp.theme.*

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun UserFormSheet(
    initial:   User?,
    formState: UserFormState,
    onSave:    (UserPayload) -> Unit,
    onDismiss: () -> Unit,
) {
    val isEdit = initial != null

    var username  by remember { mutableStateOf(initial?.username  ?: "") }
    var email     by remember { mutableStateOf(initial?.email     ?: "") }
    var firstName by remember { mutableStateOf(initial?.firstName ?: "") }
    var lastName  by remember { mutableStateOf(initial?.lastName  ?: "") }
    var password  by remember { mutableStateOf("") }
    var isStaff   by remember { mutableStateOf(initial?.isStaff  ?: false) }
    var isActive  by remember { mutableStateOf(initial?.isActive  ?: true) }

    val isSaving     = formState is UserFormState.Saving
    val usernameError= username.isNotEmpty() && username.length < 3
    val emailError   = email.isNotEmpty() && !email.contains("@")
    val passwordError= !isEdit && password.isNotEmpty() && password.length < 8
    val canSave      = username.length >= 3 && email.contains("@") &&
                       (isEdit || password.length >= 8) && !isSaving

    LaunchedEffect(formState) {
        if (formState is UserFormState.Success) onDismiss()
    }

    ModalBottomSheet(
        onDismissRequest = { if (!isSaving) onDismiss() },
        containerColor   = Surface,
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .verticalScroll(rememberScrollState())
                .padding(horizontal = 24.dp)
                .padding(bottom = 32.dp)
                .navigationBarsPadding(),
            verticalArrangement = Arrangement.spacedBy(14.dp),
        ) {
            Text(
                text       = if (isEdit) "Editar: ${initial?.username}" else "Nuevo usuario",
                style      = MaterialTheme.typography.titleLarge,
                fontWeight = FontWeight.Bold,
                color      = TextPrimary,
            )

            // Error global
            if (formState is UserFormState.Error) {
                Surface(
                    color    = Error.copy(alpha = 0.1f),
                    shape    = MaterialTheme.shapes.small,
                    modifier = Modifier.fillMaxWidth(),
                ) {
                    Text(
                        formState.message,
                        color    = Error,
                        style    = MaterialTheme.typography.bodySmall,
                        modifier = Modifier.padding(12.dp),
                    )
                }
            }

            // Usuario y Email en fila
            Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                ShopTextField(
                    value         = username,
                    onValueChange = { username = it },
                    label         = "Usuario *",
                    placeholder   = "mínimo 3 caracteres",
                    isError       = usernameError,
                    errorMessage  = "Mínimo 3 caracteres",
                    enabled       = !isSaving,
                    modifier      = Modifier.weight(1f),
                )
                ShopTextField(
                    value         = email,
                    onValueChange = { email = it },
                    label         = "Email *",
                    placeholder   = "tu@email.com",
                    isError       = emailError,
                    errorMessage  = "Email inválido",
                    enabled       = !isSaving,
                    keyboardType  = KeyboardType.Email,
                    modifier      = Modifier.weight(1f),
                )
            }

            // Nombre y Apellido en fila
            Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                ShopTextField(
                    value         = firstName,
                    onValueChange = { firstName = it },
                    label         = "Nombre",
                    placeholder   = "Juan",
                    enabled       = !isSaving,
                    modifier      = Modifier.weight(1f),
                )
                ShopTextField(
                    value         = lastName,
                    onValueChange = { lastName = it },
                    label         = "Apellido",
                    placeholder   = "Pérez",
                    enabled       = !isSaving,
                    modifier      = Modifier.weight(1f),
                )
            }

            // Contraseña
            ShopTextField(
                value         = password,
                onValueChange = { password = it },
                label         = if (isEdit) "Nueva contraseña (vacío = no cambiar)" else "Contraseña *",
                placeholder   = "mínimo 8 caracteres",
                isPassword    = true,
                isError       = passwordError,
                errorMessage  = "Mínimo 8 caracteres",
                enabled       = !isSaving,
                keyboardType  = KeyboardType.Password,
            )

            // Toggles Staff y Activo
            Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                ToggleCard(
                    label       = "Rol Staff",
                    description = "Acceso al admin",
                    checked     = isStaff,
                    onChanged   = { isStaff = it },
                    enabled     = !isSaving,
                    modifier    = Modifier.weight(1f),
                )
                ToggleCard(
                    label       = "Activo",
                    description = "Puede iniciar sesión",
                    checked     = isActive,
                    onChanged   = { isActive = it },
                    enabled     = !isSaving,
                    modifier    = Modifier.weight(1f),
                )
            }

            // Botones
            Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                OutlinedButton(
                    onClick  = { if (!isSaving) onDismiss() },
                    enabled  = !isSaving,
                    modifier = Modifier.weight(1f).height(52.dp),
                    colors   = ButtonDefaults.outlinedButtonColors(contentColor = TextSecondary),
                    border   = ButtonDefaults.outlinedButtonBorder.copy(
                        brush = androidx.compose.ui.graphics.SolidColor(Border),
                    ),
                    shape    = MaterialTheme.shapes.medium,
                ) { Text("Cancelar") }

                Button(
                    onClick  = {
                        onSave(UserPayload(
                            username  = username.trim(),
                            email     = email.trim(),
                            firstName = firstName.trim(),
                            lastName  = lastName.trim(),
                            isStaff   = isStaff,
                            isActive  = isActive,
                            password  = password.ifBlank { null },
                        ))
                    },
                    enabled  = canSave,
                    modifier = Modifier.weight(1f).height(52.dp),
                    colors   = ButtonDefaults.buttonColors(
                        containerColor         = Accent,
                        contentColor           = AccentOnDark,
                        disabledContainerColor = Accent.copy(alpha = 0.4f),
                    ),
                    shape    = MaterialTheme.shapes.medium,
                ) {
                    if (isSaving) {
                        CircularProgressIndicator(
                            color       = AccentOnDark,
                            modifier    = Modifier.size(16.dp),
                            strokeWidth = 2.dp,
                        )
                        Spacer(Modifier.width(8.dp))
                    }
                    Text(
                        if (isSaving) "Guardando..."
                        else if (isEdit) "Guardar cambios"
                        else "Crear usuario",
                        fontWeight = FontWeight.Bold,
                    )
                }
            }
        }
    }
}

@Composable
private fun ToggleCard(
    label:       String,
    description: String,
    checked:     Boolean,
    onChanged:   (Boolean) -> Unit,
    enabled:     Boolean,
    modifier:    Modifier = Modifier,
) {
    Surface(
        color    = Surface2,
        shape    = MaterialTheme.shapes.medium,
        modifier = modifier,
    ) {
        Row(
            modifier          = Modifier
                .fillMaxWidth()
                .padding(horizontal = 12.dp, vertical = 10.dp),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically,
        ) {
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    label,
                    style      = MaterialTheme.typography.bodySmall,
                    fontWeight = FontWeight.SemiBold,
                    color      = TextPrimary,
                )
                Text(
                    description,
                    style = MaterialTheme.typography.bodySmall.copy(
                        fontSize = androidx.compose.ui.unit.TextUnit(
                            10f,
                            androidx.compose.ui.unit.TextUnitType.Sp,
                        )
                    ),
                    color = TextSecondary,
                )
            }
            Switch(
                checked         = checked,
                onCheckedChange = onChanged,
                enabled         = enabled,
                colors          = SwitchDefaults.colors(
                    checkedThumbColor    = AccentOnDark,
                    checkedTrackColor    = Accent,
                    uncheckedTrackColor  = Surface,
                    uncheckedBorderColor = Border,
                ),
            )
        }
    }
}
```

---

## 11.3 Pantalla de usuarios admin — `presentation/ui/admin/users/UsersAdminScreen.kt`

```kotlin
// presentation/ui/admin/users/UsersAdminScreen.kt
package com.shopapp.presentation.ui.admin.users

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.domain.model.User
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.components.ErrorScreen
import com.shopapp.presentation.viewmodel.UserRoleFilter
import com.shopapp.presentation.viewmodel.UsersAdminViewModel
import com.shopapp.theme.*

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun UsersAdminScreen(
    viewModel: UsersAdminViewModel = hiltViewModel(),
) {
    val state     by viewModel.state.collectAsState()
    val filtered  by viewModel.filtered.collectAsState()
    val formState by viewModel.formState.collectAsState()

    var showForm    by remember { mutableStateOf(false) }
    var editTarget  by remember { mutableStateOf<User?>(null) }
    var deleteTarget by remember { mutableStateOf<User?>(null) }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(Background),
    ) {
        // ── Header ────────────────────────────────────────────
        Surface(color = Surface, tonalElevation = 0.dp) {
            Column(modifier = Modifier.padding(16.dp)) {
                Row(
                    modifier              = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment     = Alignment.CenterVertically,
                ) {
                    Column {
                        Text(
                            "Usuarios",
                            style      = MaterialTheme.typography.headlineMedium,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                        )
                        Text(
                            "${state.total} usuarios",
                            style = MaterialTheme.typography.bodySmall,
                            color = TextSecondary,
                        )
                    }
                    Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        IconButton(onClick = viewModel::load) {
                            Icon(Icons.Default.Refresh, null, tint = TextSecondary)
                        }
                        Button(
                            onClick = { editTarget = null; showForm = true },
                            colors  = ButtonDefaults.buttonColors(
                                containerColor = Accent, contentColor = AccentOnDark,
                            ),
                            shape          = MaterialTheme.shapes.medium,
                            contentPadding = PaddingValues(horizontal = 16.dp, vertical = 10.dp),
                        ) {
                            Icon(Icons.Default.PersonAdd, null, modifier = Modifier.size(18.dp))
                            Spacer(Modifier.width(4.dp))
                            Text("Nuevo", fontWeight = FontWeight.Bold)
                        }
                    }
                }

                Spacer(Modifier.height(12.dp))

                // Búsqueda
                OutlinedTextField(
                    value         = state.search,
                    onValueChange = viewModel::setSearch,
                    placeholder   = { Text("Buscar usuario o email...", color = TextFaint) },
                    leadingIcon   = { Icon(Icons.Default.Search, null, tint = TextSecondary) },
                    singleLine    = true,
                    modifier      = Modifier.fillMaxWidth(),
                    shape         = MaterialTheme.shapes.medium,
                    colors        = OutlinedTextFieldDefaults.colors(
                        focusedBorderColor   = Accent,
                        unfocusedBorderColor = Border,
                        cursorColor          = Accent,
                    ),
                )

                Spacer(Modifier.height(10.dp))

                // Filtros de rol
                LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                    items(UserRoleFilter.entries) { filter ->
                        FilterChip(
                            selected = state.roleFilter == filter,
                            onClick  = { viewModel.setRoleFilter(filter) },
                            label    = { Text(filter.label, style = MaterialTheme.typography.labelSmall) },
                            colors   = FilterChipDefaults.filterChipColors(
                                selectedContainerColor = Accent,
                                selectedLabelColor     = AccentOnDark,
                                containerColor         = Surface2,
                                labelColor             = TextSecondary,
                            ),
                        )
                    }
                }
            }
        }

        // ── Contenido ─────────────────────────────────────────
        when {
            state.isLoading -> LoadingScreen("Cargando usuarios...")
            state.error != null -> ErrorScreen(state.error!!, onRetry = viewModel::load)
            filtered.isEmpty() -> {
                Box(Modifier.fillMaxSize(), Alignment.Center) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Text("👤", fontSize = 48.sp)
                        Spacer(Modifier.height(12.dp))
                        Text(
                            if (state.search.isBlank() && state.roleFilter == UserRoleFilter.ALL)
                                "Sin usuarios" else "Sin resultados",
                            style      = MaterialTheme.typography.titleMedium,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                        )
                    }
                }
            }
            else -> {
                LazyColumn(
                    modifier       = Modifier.fillMaxSize(),
                    contentPadding = PaddingValues(16.dp),
                    verticalArrangement = Arrangement.spacedBy(10.dp),
                ) {
                    items(filtered, key = { it.id }) { user ->
                        UserAdminCard(
                            user         = user,
                            onToggleStaff  = { viewModel.toggleStaff(user.id, !user.isStaff) },
                            onToggleActive = { viewModel.toggleActive(user.id) },
                            onEdit         = { editTarget = user; showForm = true },
                            onDelete       = { deleteTarget = user },
                        )
                    }
                }
            }
        }
    }

    // Bottom Sheet formulario
    if (showForm) {
        UserFormSheet(
            initial   = editTarget,
            formState = formState,
            onSave    = { payload ->
                if (editTarget != null) viewModel.updateUser(editTarget!!.id, payload)
                else viewModel.createUser(payload)
            },
            onDismiss = {
                showForm   = false
                editTarget = null
                viewModel.resetFormState()
            },
        )
    }

    // Diálogo de eliminación
    deleteTarget?.let { user ->
        AlertDialog(
            onDismissRequest = { deleteTarget = null },
            containerColor   = Surface,
            shape            = MaterialTheme.shapes.large,
            title            = { Text("¿Eliminar usuario?", color = TextPrimary) },
            text             = {
                Text(
                    "\"${user.username}\" se eliminará permanentemente. Esta acción no se puede deshacer.",
                    color = TextSecondary,
                )
            },
            confirmButton    = {
                TextButton(onClick = {
                    viewModel.deleteUser(user.id)
                    deleteTarget = null
                }) { Text("Eliminar", color = Error, fontWeight = FontWeight.Bold) }
            },
            dismissButton    = {
                TextButton(onClick = { deleteTarget = null }) {
                    Text("Cancelar", color = TextSecondary)
                }
            },
        )
    }
}

// ── UserAdminCard ─────────────────────────────────────────────

@Composable
private fun UserAdminCard(
    user:           User,
    onToggleStaff:  () -> Unit,
    onToggleActive: () -> Unit,
    onEdit:         () -> Unit,
    onDelete:       () -> Unit,
) {
    Surface(
        shape  = MaterialTheme.shapes.large,
        color  = if (user.isActive) Surface else Surface.copy(alpha = 0.55f),
        modifier = Modifier.fillMaxWidth(),
    ) {
        Row(
            modifier          = Modifier.padding(12.dp),
            verticalAlignment = Alignment.CenterVertically,
        ) {
            // Avatar
            Box(
                modifier         = Modifier
                    .size(46.dp)
                    .background(
                        brush = if (user.isStaff)
                            androidx.compose.ui.graphics.Brush.linearGradient(
                                listOf(Accent, AccentLight)
                            )
                        else
                            androidx.compose.ui.graphics.Brush.linearGradient(
                                listOf(Surface2, Border)
                            ),
                        shape = CircleShape,
                    ),
                contentAlignment = Alignment.Center,
            ) {
                Text(
                    text       = user.username.firstOrNull()?.uppercaseChar()?.toString() ?: "?",
                    color      = if (user.isStaff) AccentOnDark else TextSecondary,
                    fontWeight = FontWeight.Bold,
                    fontSize   = 18.sp,
                )
            }

            Spacer(Modifier.width(12.dp))

            // Info
            Column(modifier = Modifier.weight(1f)) {
                Row(
                    verticalAlignment     = Alignment.CenterVertically,
                    horizontalArrangement = Arrangement.spacedBy(6.dp),
                ) {
                    Text(
                        text       = user.username,
                        style      = MaterialTheme.typography.bodyLarge,
                        fontWeight = FontWeight.SemiBold,
                        color      = TextPrimary,
                    )
                    // Badge staff
                    if (user.isStaff) {
                        Surface(
                            color  = Accent.copy(alpha = 0.15f),
                            shape  = MaterialTheme.shapes.extraSmall,
                        ) {
                            Text(
                                "Staff",
                                color      = Accent,
                                fontSize   = 10.sp,
                                fontWeight = FontWeight.Bold,
                                modifier   = Modifier.padding(horizontal = 6.dp, vertical = 2.dp),
                            )
                        }
                    }
                    // Badge inactivo
                    if (!user.isActive) {
                        Surface(
                            color  = Error.copy(alpha = 0.12f),
                            shape  = MaterialTheme.shapes.extraSmall,
                        ) {
                            Text(
                                "Inactivo",
                                color      = Error,
                                fontSize   = 10.sp,
                                fontWeight = FontWeight.Bold,
                                modifier   = Modifier.padding(horizontal = 6.dp, vertical = 2.dp),
                            )
                        }
                    }
                }
                Text(
                    text  = user.email,
                    style = MaterialTheme.typography.bodySmall,
                    color = TextSecondary,
                )
                Text(
                    text  = "${user.numOrders} pedido${if (user.numOrders != 1) "s" else ""}",
                    style = MaterialTheme.typography.bodySmall,
                    color = Accent,
                    fontWeight = FontWeight.SemiBold,
                )
            }

            // Acciones
            Column(
                horizontalAlignment = Alignment.End,
                verticalArrangement = Arrangement.spacedBy(4.dp),
            ) {
                Row(horizontalArrangement = Arrangement.spacedBy(2.dp)) {
                    // Toggle staff
                    IconButton(onClick = onToggleStaff, modifier = Modifier.size(32.dp)) {
                        Icon(
                            imageVector = if (user.isStaff) Icons.Default.AdminPanelSettings
                                          else Icons.Default.PersonOutline,
                            contentDescription = if (user.isStaff) "Quitar staff" else "Dar staff",
                            tint     = if (user.isStaff) Accent else TextFaint,
                            modifier = Modifier.size(18.dp),
                        )
                    }
                    // Toggle activo
                    IconButton(onClick = onToggleActive, modifier = Modifier.size(32.dp)) {
                        Icon(
                            imageVector = if (user.isActive) Icons.Default.ToggleOn
                                          else Icons.Default.ToggleOff,
                            contentDescription = if (user.isActive) "Desactivar" else "Activar",
                            tint     = if (user.isActive) Success else TextFaint,
                            modifier = Modifier.size(22.dp),
                        )
                    }
                    // Editar
                    IconButton(onClick = onEdit, modifier = Modifier.size(32.dp)) {
                        Icon(
                            Icons.Default.Edit, contentDescription = "Editar",
                            tint = TextSecondary, modifier = Modifier.size(18.dp),
                        )
                    }
                    // Eliminar
                    IconButton(onClick = onDelete, modifier = Modifier.size(32.dp)) {
                        Icon(
                            Icons.Default.PersonRemove, contentDescription = "Eliminar",
                            tint = Error, modifier = Modifier.size(18.dp),
                        )
                    }
                }
            }
        }
    }
}
```

---

## 11.4 Actualizar NavGraph — ruta `admin/users` y router final

```kotlin
// presentation/navigation/NavGraph.kt
package com.shopapp.presentation.navigation

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.navigation.*
import androidx.navigation.compose.*
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.ui.admin.AdminScaffold
import com.shopapp.presentation.ui.admin.categories.CategoriesAdminScreen
import com.shopapp.presentation.ui.admin.dashboard.DashboardScreen
import com.shopapp.presentation.ui.admin.orders.OrderAdminDetailScreen
import com.shopapp.presentation.ui.admin.orders.OrdersAdminScreen
import com.shopapp.presentation.ui.admin.products.ProductsAdminScreen
import com.shopapp.presentation.ui.admin.users.UsersAdminScreen
import com.shopapp.presentation.ui.auth.LoginScreen
import com.shopapp.presentation.ui.auth.RegisterScreen
import com.shopapp.presentation.ui.client.orders.OrderDetailScreen
import com.shopapp.presentation.ui.client.orders.OrdersScreen
import com.shopapp.presentation.ui.client.profile.ProfileScreen
import com.shopapp.presentation.ui.uipublic.cart.CartBottomSheet
import com.shopapp.presentation.ui.uipublic.catalog.CatalogScreen
import com.shopapp.presentation.ui.uipublic.home.HomeScreen
import com.shopapp.presentation.ui.uipublic.product.ProductDetailScreen
import com.shopapp.presentation.viewmodel.AuthViewModel
import com.shopapp.presentation.viewmodel.CartViewModel
import com.shopapp.presentation.viewmodel.OrdersAdminViewModel
import com.shopapp.theme.Surface
import com.shopapp.theme.TextSecondary

@Composable
fun NavGraph(
    authViewModel: AuthViewModel,
    cartViewModel: CartViewModel = hiltViewModel(),
) {
    val navController     = rememberNavController()
    val isCheckingSession by authViewModel.isCheckingSession.collectAsState()
    val isAuthenticated   by authViewModel.isAuthenticated.collectAsState()
    val isStaff           by authViewModel.isStaff.collectAsState()
    val cartCount         by cartViewModel.totalItems.collectAsState()
    val currentUser       by authViewModel.currentUser.collectAsState()

    var showCart         by remember { mutableStateOf(false) }
    var confirmedOrderId by remember { mutableStateOf<Int?>(null) }

    if (isCheckingSession) {
        LoadingScreen("Iniciando ShopApp...")
        return
    }

    val startDestination = when {
        !isAuthenticated -> Screen.Login.route
        isStaff          -> Screen.AdminDashboard.route
        else             -> Screen.Home.route
    }

    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute      = navBackStackEntry?.destination?.route

    val showBottomBar = currentRoute in listOf(
        Screen.Home.route,
        Screen.Catalog.route,
        Screen.Orders.route,
        Screen.Profile.route,
    )

    Scaffold(
        containerColor = Surface,
        bottomBar = {
            if (showBottomBar) {
                BottomNavBar(
                    navController = navController,
                    cartCount     = cartCount,
                    onCartClick   = { showCart = true },
                )
            }
        },
    ) { innerPadding ->

        // ── BottomSheet carrito
        if (showCart) {
            CartBottomSheet(
                cartViewModel   = cartViewModel,
                isAuthenticated = isAuthenticated,
                onDismiss       = { showCart = false },
                onLoginRequired = {
                    showCart = false
                    navController.navigate(Screen.Login.route)
                },
                onOrderSuccess = { orderId ->
                    confirmedOrderId = orderId
                    showCart = false
                },
            )
        }

        NavHost(
            navController    = navController,
            startDestination = startDestination,
            modifier         = Modifier.padding(innerPadding),
        ) {

            // ── LOGIN ───────────────────────────────
            composable(Screen.Login.route) {
                LoginScreen(
                    onLoginSuccess = { staff ->
                        val dest = if (staff) Screen.AdminDashboard.route else Screen.Home.route
                        navController.navigate(dest) {
                            popUpTo(Screen.Login.route) { inclusive = true }
                        }
                    },
                    onNavigateToRegister = { navController.navigate(Screen.Register.route) },
                    viewModel            = authViewModel,
                )
            }

            // ── REGISTER ────────────────────────────
            composable(Screen.Register.route) {
                RegisterScreen(
                    onRegisterSuccess = { staff ->
                        val dest = if (staff) Screen.AdminDashboard.route else Screen.Home.route
                        navController.navigate(dest) {
                            popUpTo(Screen.Login.route) { inclusive = true }
                        }
                    },
                    onNavigateToLogin = { navController.popBackStack() },
                    viewModel         = authViewModel,
                )
            }

            // ── HOME ───────────────────────────────
            composable(Screen.Home.route) {
                HomeScreen(
                    onProductClick = { id -> navController.navigate("product/$id") },
                    onCatalogClick = { navController.navigate(Screen.Catalog.route) },
                )
            }

            // ── CATALOGO ───────────────────────────
            composable(Screen.Catalog.route) {
                CatalogScreen(
                    onProductClick = { id -> navController.navigate("product/$id") },
                )
            }

            // ── DETALLE PRODUCTO ───────────────────
            composable(
                route     = "product/{id}",
                arguments = listOf(navArgument("id") { type = NavType.IntType }),
            ) { backStackEntry ->
                val id = backStackEntry.arguments?.getInt("id") ?: return@composable
                ProductDetailScreen(
                    productId     = id,
                    onBack        = { navController.popBackStack() },
                    cartViewModel = cartViewModel,
                )
            }

            // ── ORDERS CLIENT ──────────────────────
            composable(Screen.Orders.route) {
                if (!isAuthenticated) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Login.route) {
                            popUpTo(Screen.Home.route)
                        }
                    }
                } else {
                    OrdersScreen(
                        onOrderClick = { id -> navController.navigate("orders/$id") },
                    )
                }
            }

            // ── ORDER DETAIL CLIENT ────────────────
            composable(
                route     = "orders/{id}",
                arguments = listOf(navArgument("id") { type = NavType.IntType }),
            ) { backStackEntry ->
                val id = backStackEntry.arguments?.getInt("id") ?: return@composable
                OrderDetailScreen(
                    orderId = id,
                    onBack  = { navController.popBackStack() },
                )
            }

            // ── PROFILE ────────────────────────────
            composable(Screen.Profile.route) {
                if (!isAuthenticated) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Login.route) {
                            popUpTo(Screen.Home.route)
                        }
                    }
                } else {
                    ProfileScreen(
                        authViewModel = authViewModel,
                        onLogout = {
                            navController.navigate(Screen.Login.route) {
                                popUpTo(0) { inclusive = true }
                            }
                        },
                    )
                }
            }

            // ── ADMIN DASHBOARD ────────────────────
            composable(Screen.AdminDashboard.route) {
                if (!isStaff) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Home.route) { popUpTo(0) }
                    }
                    return@composable
                }

                AdminScaffold(
                    currentRoute = Screen.AdminDashboard.route,
                    user         = currentUser,
                    title        = "Dashboard",
                    onNavClick   = { route ->
                        navController.navigate(route) {
                            launchSingleTop = true
                            restoreState    = true
                        }
                    },
                    onStoreClick = { navController.navigate(Screen.Home.route) },
                    onLogout     = {
                        authViewModel.logout()
                        navController.navigate(Screen.Login.route) {
                            popUpTo(0) { inclusive = true }
                        }
                    },
                ) { padding ->
                    Box(modifier = Modifier.padding(padding)) {
                        DashboardScreen(
                            onNavigate = { route -> navController.navigate(route) }
                        )
                    }
                }
            }

            // ── ADMIN CATEGORIES ───────────────────
            composable("admin/categories") {
                if (!isStaff) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Home.route) { popUpTo(0) }
                    }
                    return@composable
                }

                AdminScaffold(
                    currentRoute = "admin/categories",
                    user         = currentUser,
                    title        = "Categorías",
                    onNavClick   = { route ->
                        navController.navigate(route) { launchSingleTop = true }
                    },
                    onStoreClick = { navController.navigate(Screen.Home.route) },
                    onLogout     = {
                        authViewModel.logout()
                        navController.navigate(Screen.Login.route) {
                            popUpTo(0) { inclusive = true }
                        }
                    },
                ) { padding ->
                    Box(modifier = Modifier.padding(padding)) {
                        CategoriesAdminScreen()
                    }
                }
            }

            // ── ADMIN PRODUCTS ─────────────────────
            composable("admin/products") {
                if (!isStaff) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Home.route) { popUpTo(0) }
                    }
                    return@composable
                }

                AdminScaffold(
                    currentRoute = "admin/products",
                    user         = currentUser,
                    title        = "Productos",
                    onNavClick   = { route ->
                        navController.navigate(route) { launchSingleTop = true }
                    },
                    onStoreClick = { navController.navigate(Screen.Home.route) },
                    onLogout     = {
                        authViewModel.logout()
                        navController.navigate(Screen.Login.route) {
                            popUpTo(0) { inclusive = true }
                        }
                    },
                ) { padding ->
                    Box(modifier = Modifier.padding(padding)) {
                        ProductsAdminScreen()
                    }
                }
            }

            // ── ADMIN ORDERS ───────────────────────
            composable("admin/orders") {
                if (!isStaff) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Home.route) { popUpTo(0) }
                    }
                    return@composable
                }

                val ordersAdminVm: OrdersAdminViewModel = hiltViewModel()

                AdminScaffold(
                    currentRoute = "admin/orders",
                    user         = currentUser,
                    title        = "Pedidos",
                    onNavClick   = { route ->
                        navController.navigate(route) { launchSingleTop = true }
                    },
                    onStoreClick = { navController.navigate(Screen.Home.route) },
                    onLogout     = {
                        authViewModel.logout()
                        navController.navigate(Screen.Login.route) {
                            popUpTo(0) { inclusive = true }
                        }
                    },
                ) { padding ->
                    Box(modifier = Modifier.padding(padding)) {
                        OrdersAdminScreen(
                            onOrderDetail = { id ->
                                navController.navigate("admin/orders/$id")
                            },
                            viewModel = ordersAdminVm,
                        )
                    }
                }
            }

            // ── ADMIN ORDER DETAIL ─────────────────
            composable(
                route     = "admin/orders/{id}",
                arguments = listOf(navArgument("id") { type = NavType.IntType }),
            ) { backStackEntry ->
                val id = backStackEntry.arguments?.getInt("id") ?: return@composable

                if (!isStaff) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Home.route) { popUpTo(0) }
                    }
                    return@composable
                }

                val parentEntry = remember(backStackEntry) {
                    navController.getBackStackEntry("admin/orders")
                }

                val ordersAdminVm: OrdersAdminViewModel = hiltViewModel(parentEntry)

                AdminScaffold(
                    currentRoute = "admin/orders",
                    user         = currentUser,
                    title        = "Detalle pedido #$id",
                    onNavClick   = { route ->
                        navController.navigate(route) { launchSingleTop = true }
                    },
                    onStoreClick = { navController.navigate(Screen.Home.route) },
                    onLogout     = {
                        authViewModel.logout()
                        navController.navigate(Screen.Login.route) {
                            popUpTo(0) { inclusive = true }
                        }
                    },
                ) { padding ->
                    Box(modifier = Modifier.padding(padding)) {
                        OrderAdminDetailScreen(
                            orderId = id,
                            onBack  = { navController.popBackStack() },
                            onStatusChange = { ordId, newStatus ->
                                ordersAdminVm.changeStatus(ordId, newStatus)
                            },
                        )
                    }
                }
            }

            // ── ADMIN USERS (CORREGIDO) ────────────
            composable("admin/users") {
                if (!isStaff) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Home.route) { popUpTo(0) }
                    }
                    return@composable
                }

                AdminScaffold(
                    currentRoute = "admin/users",
                    user         = currentUser,
                    title        = "Usuarios",
                    onNavClick   = { route ->
                        navController.navigate(route) { launchSingleTop = true }
                    },
                    onStoreClick = { navController.navigate(Screen.Home.route) },
                    onLogout     = {
                        authViewModel.logout()
                        navController.navigate(Screen.Login.route) {
                            popUpTo(0) { inclusive = true }
                        }
                    },
                ) { padding ->
                    Box(modifier = Modifier.padding(padding)) {
                        UsersAdminScreen()
                    }
                }
            }
        }
    }
}


```

---

## 11.5 Resumen completo del tutorial

El tutorial de 12 módulos está completo. Verificar que el NavGraph tiene todas las rutas:

```kotlin
// NavGraph.kt — rutas finales
NavHost(navController, startDestination) {
    // Auth
    composable("login")    { LoginScreen(...)    }
    composable("register") { RegisterScreen(...) }

    // Público con BottomBar
    composable("home")           { HomeScreen(...)          }
    composable("catalog")        { CatalogScreen(...)       }
    composable("product/{id}")   { ProductDetailScreen(...) }
    composable("cart")           { CartBottomSheet(...)     } // controlado con estado local

    // Cliente privado
    composable("orders")        { OrdersScreen(...)       }
    composable("orders/{id}")   { OrderDetailScreen(...)  }
    composable("profile")       { ProfileScreen(...)      }

    // Admin
    composable("admin")                { DashboardScreen(...)           }
    composable("admin/categories")     { CategoriesAdminScreen(...)     }
    composable("admin/products")       { ProductsAdminScreenWrapper()   }
    composable("admin/orders")         { OrdersAdminScreen(...)         }
    composable("admin/orders/{id}")    { OrderAdminDetailScreen(...)    }
    composable("admin/users")          { UsersAdminScreen()             }
}
```

---

## ✅ Checkpoint Módulo 11

### Crear usuario de prueba para verificar el flujo staff

```bash
# En Django — verificar que el usuario tiene is_staff=True
uv run python manage.py shell -c "
from django.contrib.auth.models import User
u = User.objects.get(username='admin')
print(f'{u.username} staff={u.is_staff} active={u.is_active}')
"
```

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Drawer → "Usuarios" | Lista con avatares dorados (staff) y grises (cliente) |
| 2 | Filtro "Staff" | Solo usuarios con rol staff |
| 3 | Filtro "Inactivos" | Usuarios con opacidad reducida |
| 4 | Buscar "admin" | Filtrado local instantáneo |
| 5 | Ícono 🛡️ toggle staff | Avatar cambia a dorado, badge "Staff" aparece |
| 6 | Ícono toggle activo | Badge "Inactivo" aparece, opacidad reduce |
| 7 | Botón "Nuevo" | Bottom sheet con formulario completo |
| 8 | Email sin @ | Botón deshabilitado |
| 9 | Contraseña < 8 al crear | Campo en rojo con error |
| 10 | Crear usuario con is_staff=true | Badge dorado en la lista |
| 11 | Nuevo staff hace login | Redirige a `/admin` automáticamente |
| 12 | Contraseña vacía al editar | No cambia la contraseña (campo opcional) |

---

## Resumen del tutorial completo — 12 módulos

| Módulo | Contenido | Checkpoint |
|--------|-----------|------------|
| M1  | Setup + Material 3 + tipos de dominio | Pantalla de verificación en emulador |
| M2  | Retrofit + interceptores JWT + DTOs | Categorías del backend en pantalla |
| M3  | AuthViewModel + Login + Registro | Login/logout funcional con JWT |
| M4  | NavGraph + CartViewModel + Home + Catálogo | Navegar y filtrar productos |
| M5  | Detalle + CartBottomSheet + Checkout | Pedido real confirmado en Django |
| M6  | Mis pedidos + progreso + Perfil + Logout | Ver historial y cerrar sesión |
| M7  | NavigationDrawer + Dashboard con KPIs | Métricas reales del backend |
| M8  | Admin CRUD Categorías | Crear, editar, toggle optimista |
| M9  | Admin CRUD Productos + restock | Producto con badge de stock |
| M10 | Admin CRUD Pedidos + estado inline | Estado actualizado en vista cliente |
| M11 | Admin CRUD Usuarios + roles | Nuevo staff accede al panel admin |

### Endpoints consumidos — resumen

```
POST   /api/auth/login/                  → M3
POST   /api/auth/register/               → M3
POST   /api/auth/logout/                 → M6
POST   /api/auth/token/refresh/          → M2 (interceptor)
GET    /api/categories/                  → M4, M8
POST   /api/categories/                  → M8
PATCH  /api/categories/{id}/             → M8
DELETE /api/categories/{id}/             → M8
GET    /api/categories/stats/            → M7
GET    /api/products/                    → M4, M9
GET    /api/products/{id}/               → M5
POST   /api/products/                    → M9
PATCH  /api/products/{id}/              → M9
DELETE /api/products/{id}/              → M9
POST   /api/products/{id}/restock/       → M9
GET    /api/products/stats/              → M7
POST   /api/orders/                      → M5
POST   /api/orders/{id}/add-item/        → M5
POST   /api/orders/{id}/confirm/         → M5
GET    /api/orders/                      → M6, M10
GET    /api/orders/{id}/                 → M6, M10
POST   /api/orders/{id}/update-status/   → M10
GET    /api/orders/stats/                → M7
GET    /api/users/                       → M11
POST   /api/users/                       → M11
PATCH  /api/users/{id}/                  → M11
DELETE /api/users/{id}/                  → M11
POST   /api/users/{id}/toggle-active/    → M11
GET    /api/users/stats/                 → M7
```