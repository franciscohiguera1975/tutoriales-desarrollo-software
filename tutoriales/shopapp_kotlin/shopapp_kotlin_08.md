# ShopApp Android — Módulo 8
## Admin CRUD Categorías — Lista, crear, editar y toggle optimista

> **Objetivo:** CRUD completo de categorías desde el panel admin: lista con búsqueda local, toggle de estado optimista, bottom sheet de formulario con validación y slug auto-generado.
> **Checkpoint final:** crear, editar y activar/desactivar categorías — verificar los cambios en el catálogo público de la app.

---

## 8.1 ViewModel de categorías admin — `presentation/viewmodel/CategoriesAdminViewModel.kt`

```kotlin
// presentation/viewmodel/CategoriesAdminViewModel.kt
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.Category
import com.shopapp.domain.model.CategoryPayload
import com.shopapp.domain.repository.CategoryRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

data class CategoriesAdminUiState(
    val categories: List<Category> = emptyList(),
    val isLoading:  Boolean        = false,
    val error:      String?        = null,
    val search:     String         = "",
)

sealed interface CategoryFormState {
    data object Idle                       : CategoryFormState
    data object Saving                     : CategoryFormState
    data class  Success(val msg: String)   : CategoryFormState
    data class  Error(val message: String) : CategoryFormState
}

@HiltViewModel
class CategoriesAdminViewModel @Inject constructor(
    private val repository: CategoryRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(CategoriesAdminUiState())
    val state: StateFlow<CategoriesAdminUiState> = _state.asStateFlow()

    private val _formState = MutableStateFlow<CategoryFormState>(CategoryFormState.Idle)
    val formState: StateFlow<CategoryFormState> = _formState.asStateFlow()

    val filtered: StateFlow<List<Category>> = _state
        .map { s ->
            if (s.search.isBlank()) s.categories
            else s.categories.filter {
                it.name.contains(s.search, ignoreCase = true)
            }
        }
        .stateIn(viewModelScope, SharingStarted.Eagerly, emptyList())

    init { load() }

    fun load() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            repository.getCategories()
                .onSuccess { cats ->
                    _state.update { it.copy(categories = cats, isLoading = false) }
                }
                .onFailure { e ->
                    _state.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    fun setSearch(query: String) {
        _state.update { it.copy(search = query) }
    }

    // Toggle optimista
    fun toggleActive(id: Int, isActive: Boolean) {
        _state.update { s ->
            s.copy(categories = s.categories.map {
                if (it.id == id) it.copy(isActive = isActive) else it
            })
        }
        viewModelScope.launch {
            repository.updateCategory(id, getCategoryPayload(id, isActive))
                .onFailure {
                    // Revertir
                    _state.update { s ->
                        s.copy(categories = s.categories.map { cat ->
                            if (cat.id == id) cat.copy(isActive = !isActive) else cat
                        })
                    }
                }
        }
    }

    private fun getCategoryPayload(id: Int, isActive: Boolean): CategoryPayload {
        val cat = _state.value.categories.first { it.id == id }
        return CategoryPayload(cat.name, cat.slug, cat.description, isActive)
    }

    fun createCategory(payload: CategoryPayload) {
        _formState.value = CategoryFormState.Saving
        viewModelScope.launch {
            repository.createCategory(payload)
                .onSuccess { created ->
                    _state.update { s ->
                        s.copy(categories = listOf(created) + s.categories)
                    }
                    _formState.value = CategoryFormState.Success("Categoría creada")
                }
                .onFailure { e ->
                    _formState.value = CategoryFormState.Error(e.message ?: "Error al crear")
                }
        }
    }

    fun updateCategory(id: Int, payload: CategoryPayload) {
        _formState.value = CategoryFormState.Saving
        viewModelScope.launch {
            repository.updateCategory(id, payload)
                .onSuccess { updated ->
                    _state.update { s ->
                        s.copy(categories = s.categories.map {
                            if (it.id == id) updated else it
                        })
                    }
                    _formState.value = CategoryFormState.Success("Categoría actualizada")
                }
                .onFailure { e ->
                    _formState.value = CategoryFormState.Error(e.message ?: "Error al actualizar")
                }
        }
    }

    fun deleteCategory(id: Int) {
        viewModelScope.launch {
            repository.deleteCategory(id)
                .onSuccess {
                    _state.update { s ->
                        s.copy(categories = s.categories.filter { it.id != id })
                    }
                }
                .onFailure { e ->
                    _state.update { it.copy(error = e.message) }
                }
        }
    }

    fun resetFormState() { _formState.value = CategoryFormState.Idle }
}
```

---

## 8.2 Formulario de categoría — `presentation/ui/admin/categories/CategoryFormSheet.kt`

```kotlin
// presentation/ui/admin/categories/CategoryFormSheet.kt
package com.shopapp.presentation.ui.admin.categories

import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import com.shopapp.domain.model.Category
import com.shopapp.domain.model.CategoryPayload
import com.shopapp.presentation.components.ShopButton
import com.shopapp.presentation.components.ShopTextField
import com.shopapp.presentation.viewmodel.CategoryFormState
import com.shopapp.theme.*

fun String.toSlug(): String = this
    .lowercase()
    .replace(Regex("[áàäâã]"), "a")
    .replace(Regex("[éèëê]"), "e")
    .replace(Regex("[íìïî]"), "i")
    .replace(Regex("[óòöôõ]"), "o")
    .replace(Regex("[úùüû]"), "u")
    .replace(Regex("[ñ]"), "n")
    .replace(Regex("[^a-z0-9\\s-]"), "")
    .trim()
    .replace(Regex("\\s+"), "-")

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CategoryFormSheet(
    initial:    Category?,
    formState:  CategoryFormState,
    onSave:     (CategoryPayload) -> Unit,
    onDismiss:  () -> Unit,
) {
    val isEdit = initial != null

    var name        by remember { mutableStateOf(initial?.name        ?: "") }
    var slug        by remember { mutableStateOf(initial?.slug        ?: "") }
    var description by remember { mutableStateOf(initial?.description ?: "") }
    var isActive    by remember { mutableStateOf(initial?.isActive    ?: true) }
    var slugEdited  by remember { mutableStateOf(isEdit) }

    // Auto-generar slug desde el nombre si no se ha editado manualmente
    LaunchedEffect(name) {
        if (!slugEdited) slug = name.toSlug()
    }

    // Cerrar al éxito
    LaunchedEffect(formState) {
        if (formState is CategoryFormState.Success) onDismiss()
    }

    val isSaving = formState is CategoryFormState.Saving
    val nameError = name.isNotEmpty() && name.length < 2
    val slugError = slug.isNotEmpty() && !slug.matches(Regex("[a-z0-9-]+"))
    val canSave   = name.length >= 2 && slug.isNotEmpty() && !nameError && !slugError && !isSaving

    ModalBottomSheet(
        onDismissRequest = { if (!isSaving) onDismiss() },
        containerColor   = Surface,
        dragHandle       = {
            Box(
                modifier         = Modifier
                    .padding(vertical = 12.dp)
                    .size(40.dp, 4.dp)
                    .then(Modifier)
                    .padding(0.dp),
                contentAlignment = Alignment.Center,
            ) {
                Surface(
                    modifier = Modifier.size(40.dp, 4.dp),
                    color    = Border,
                    shape    = MaterialTheme.shapes.extraSmall,
                ) {}
            }
        },
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .verticalScroll(rememberScrollState())
                .padding(horizontal = 24.dp)
                .padding(bottom = 32.dp)
                .navigationBarsPadding(),
            verticalArrangement = Arrangement.spacedBy(16.dp),
        ) {
            // Título
            Text(
                text       = if (isEdit) "Editar: ${initial?.name}" else "Nueva categoría",
                style      = MaterialTheme.typography.titleLarge,
                fontWeight = FontWeight.Bold,
                color      = TextPrimary,
            )

            // Error del formulario
            if (formState is CategoryFormState.Error) {
                Surface(
                    color  = Error.copy(alpha = 0.1f),
                    shape  = MaterialTheme.shapes.small,
                    modifier = Modifier.fillMaxWidth(),
                ) {
                    Text(
                        text     = formState.message,
                        color    = Error,
                        style    = MaterialTheme.typography.bodySmall,
                        modifier = Modifier.padding(12.dp),
                    )
                }
            }

            // Nombre
            ShopTextField(
                value         = name,
                onValueChange = { name = it },
                label         = "Nombre *",
                placeholder   = "ej. Electrónica",
                isError       = nameError,
                errorMessage  = "Mínimo 2 caracteres",
                enabled       = !isSaving,
            )

            // Slug
            ShopTextField(
                value         = slug,
                onValueChange = { slug = it; slugEdited = true },
                label         = "Slug (URL) *",
                placeholder   = "ej. electronica",
                isError       = slugError,
                errorMessage  = "Solo minúsculas, números y guiones",
                enabled       = !isSaving,
            )
            Text(
                text  = "URL: /catalog?category=$slug",
                style = MaterialTheme.typography.bodySmall,
                color = TextFaint,
            )

            // Descripción
            OutlinedTextField(
                value         = description,
                onValueChange = { description = it },
                label         = { Text("Descripción") },
                placeholder   = { Text("Descripción opcional", color = TextFaint) },
                minLines      = 3,
                maxLines      = 5,
                enabled       = !isSaving,
                modifier      = Modifier.fillMaxWidth(),
                colors        = OutlinedTextFieldDefaults.colors(
                    focusedBorderColor   = Accent,
                    focusedLabelColor    = Accent,
                    unfocusedBorderColor = Border,
                    unfocusedLabelColor  = TextSecondary,
                ),
            )

            // Toggle activa
            Surface(
                color  = Surface2,
                shape  = MaterialTheme.shapes.medium,
                modifier = Modifier.fillMaxWidth(),
            ) {
                Row(
                    modifier          = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp, vertical = 12.dp),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment = Alignment.CenterVertically,
                ) {
                    Column {
                        Text(
                            "Categoría activa",
                            style      = MaterialTheme.typography.bodyMedium,
                            fontWeight = FontWeight.SemiBold,
                            color      = TextPrimary,
                        )
                        Text(
                            "Visible en el catálogo público",
                            style = MaterialTheme.typography.bodySmall,
                            color = TextSecondary,
                        )
                    }
                    Switch(
                        checked         = isActive,
                        onCheckedChange = { isActive = it },
                        enabled         = !isSaving,
                        colors          = SwitchDefaults.colors(
                            checkedThumbColor       = AccentOnDark,
                            checkedTrackColor       = Accent,
                            uncheckedThumbColor     = TextSecondary,
                            uncheckedTrackColor     = Surface,
                            uncheckedBorderColor    = Border,
                        ),
                    )
                }
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
                ) {
                    Text("Cancelar")
                }
                Button(
                    onClick  = {
                        onSave(CategoryPayload(name.trim(), slug.trim(), description.trim(), isActive))
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
                        else "Crear categoría",
                        fontWeight = FontWeight.Bold,
                    )
                }
            }
        }
    }
}
```

---

## 8.3 Pantalla de categorías — `presentation/ui/admin/categories/CategoriesAdminScreen.kt`

```kotlin
// presentation/ui/admin/categories/CategoriesAdminScreen.kt
package com.shopapp.presentation.ui.admin.categories

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
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
import com.shopapp.domain.model.Category
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.components.ErrorScreen
import com.shopapp.presentation.viewmodel.CategoriesAdminViewModel
import com.shopapp.theme.*

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CategoriesAdminScreen(
    viewModel: CategoriesAdminViewModel = hiltViewModel(),
) {
    val state     by viewModel.state.collectAsState()
    val filtered  by viewModel.filtered.collectAsState()
    val formState by viewModel.formState.collectAsState()

    var showForm    by remember { mutableStateOf(false) }
    var editTarget  by remember { mutableStateOf<Category?>(null) }
    var deleteTarget by remember { mutableStateOf<Category?>(null) }

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
                            text       = "Categorías",
                            style      = MaterialTheme.typography.headlineMedium,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                        )
                        Text(
                            text  = "${state.categories.size} categorías",
                            style = MaterialTheme.typography.bodySmall,
                            color = TextSecondary,
                        )
                    }
                    Button(
                        onClick = { editTarget = null; showForm = true },
                        colors  = ButtonDefaults.buttonColors(
                            containerColor = Accent,
                            contentColor   = AccentOnDark,
                        ),
                        shape    = MaterialTheme.shapes.medium,
                        contentPadding = PaddingValues(horizontal = 16.dp, vertical = 10.dp),
                    ) {
                        Icon(Icons.Default.Add, contentDescription = null, modifier = Modifier.size(18.dp))
                        Spacer(Modifier.width(6.dp))
                        Text("Nueva", fontWeight = FontWeight.Bold)
                    }
                }

                Spacer(Modifier.height(12.dp))

                // Búsqueda
                OutlinedTextField(
                    value         = state.search,
                    onValueChange = viewModel::setSearch,
                    placeholder   = { Text("Buscar categoría...", color = TextFaint) },
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
            }
        }

        // ── Contenido ─────────────────────────────────────────
        when {
            state.isLoading -> LoadingScreen("Cargando categorías...")
            state.error != null -> ErrorScreen(state.error!!, onRetry = viewModel::load)
            filtered.isEmpty() -> {
                Box(Modifier.fillMaxSize(), Alignment.Center) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Text("🏷️", fontSize = 48.sp)
                        Spacer(Modifier.height(12.dp))
                        Text(
                            if (state.search.isBlank()) "Sin categorías" else "Sin resultados",
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
                    items(filtered, key = { it.id }) { category ->
                        CategoryAdminCard(
                            category  = category,
                            onToggle  = { viewModel.toggleActive(category.id, !category.isActive) },
                            onEdit    = { editTarget = category; showForm = true },
                            onDelete  = { deleteTarget = category },
                        )
                    }
                }
            }
        }
    }

    // ── Bottom Sheet de formulario ────────────────────────────
    if (showForm) {
        CategoryFormSheet(
            initial   = editTarget,
            formState = formState,
            onSave    = { payload ->
                if (editTarget != null) viewModel.updateCategory(editTarget!!.id, payload)
                else viewModel.createCategory(payload)
            },
            onDismiss = {
                showForm   = false
                editTarget = null
                viewModel.resetFormState()
            },
        )
    }

    // ── Diálogo de confirmación de eliminación ─────────────────
    deleteTarget?.let { cat ->
        AlertDialog(
            onDismissRequest = { deleteTarget = null },
            title            = {
                Text(
                    if (cat.totalProducts > 0) "¿Desactivar categoría?"
                    else "¿Eliminar categoría?",
                    color = TextPrimary,
                )
            },
            text             = {
                Text(
                    if (cat.totalProducts > 0)
                        "\"${cat.name}\" tiene ${cat.totalProducts} producto(s). Se desactivará en lugar de eliminarse."
                    else
                        "\"${cat.name}\" se eliminará permanentemente.",
                    color = TextSecondary,
                )
            },
            confirmButton    = {
                TextButton(onClick = {
                    if (cat.totalProducts > 0) viewModel.toggleActive(cat.id, false)
                    else viewModel.deleteCategory(cat.id)
                    deleteTarget = null
                }) {
                    Text(
                        if (cat.totalProducts > 0) "Desactivar" else "Eliminar",
                        color      = Error,
                        fontWeight = FontWeight.Bold,
                    )
                }
            },
            dismissButton    = {
                TextButton(onClick = { deleteTarget = null }) {
                    Text("Cancelar", color = TextSecondary)
                }
            },
            containerColor   = Surface,
            shape            = MaterialTheme.shapes.large,
        )
    }
}

// ── CategoryAdminCard ─────────────────────────────────────────

@Composable
private fun CategoryAdminCard(
    category: Category,
    onToggle: () -> Unit,
    onEdit:   () -> Unit,
    onDelete: () -> Unit,
) {
    Surface(
        shape  = MaterialTheme.shapes.large,
        color  = Surface,
        modifier = Modifier.fillMaxWidth(),
    ) {
        Row(
            modifier          = Modifier
                .fillMaxWidth()
                .padding(12.dp),
            verticalAlignment = Alignment.CenterVertically,
        ) {
            // Toggle
            Switch(
                checked         = category.isActive,
                onCheckedChange = { onToggle() },
                colors          = SwitchDefaults.colors(
                    checkedThumbColor    = AccentOnDark,
                    checkedTrackColor    = Accent,
                    uncheckedTrackColor  = Surface2,
                    uncheckedBorderColor = Border,
                ),
            )

            Spacer(Modifier.width(12.dp))

            // Info
            Column(modifier = Modifier.weight(1f)) {
                Row(
                    verticalAlignment = Alignment.CenterVertically,
                    horizontalArrangement = Arrangement.spacedBy(8.dp),
                ) {
                    Text(
                        text       = category.name,
                        style      = MaterialTheme.typography.bodyLarge,
                        fontWeight = FontWeight.SemiBold,
                        color      = TextPrimary,
                    )
                    if (!category.isActive) {
                        Surface(
                            color  = Error.copy(alpha = 0.12f),
                            shape  = MaterialTheme.shapes.extraSmall,
                        ) {
                            Text(
                                "Inactiva",
                                color    = Error,
                                fontSize = 10.sp,
                                fontWeight = FontWeight.Bold,
                                modifier = Modifier.padding(horizontal = 6.dp, vertical = 2.dp),
                            )
                        }
                    }
                }
                Text(
                    text  = "/${category.slug}",
                    style = MaterialTheme.typography.bodySmall,
                    color = TextFaint,
                )
                Text(
                    text  = "${category.totalProducts} productos",
                    style = MaterialTheme.typography.bodySmall,
                    color = Accent,
                    fontWeight = FontWeight.SemiBold,
                )
            }

            // Acciones
            Row {
                IconButton(onClick = onEdit) {
                    Icon(
                        Icons.Default.Edit,
                        contentDescription = "Editar",
                        tint = TextSecondary,
                        modifier = Modifier.size(20.dp),
                    )
                }
                IconButton(onClick = onDelete) {
                    Icon(
                        Icons.Default.Delete,
                        contentDescription = "Eliminar",
                        tint = Error,
                        modifier = Modifier.size(20.dp),
                    )
                }
            }
        }
    }
}
```

---

## 8.4 Actualizar NavGraph — reemplazar placeholder de categorías

```kotlin
// NavGraph.kt — reemplazar el placeholder "admin/categories"

// Importar:
import com.shopapp.presentation.ui.admin.categories.CategoriesAdminScreen

// Dentro del NavHost, reemplazar el composable de "admin/categories":
composable("admin/categories") {
    if (!isStaff) {
        LaunchedEffect(Unit) { navController.navigate(Screen.Home.route) { popUpTo(0) } }
        return@composable
    }
    AdminScaffold(
        currentRoute = "admin/categories",
        user         = authViewModel.currentUser.collectAsState().value,
        title        = "Categorías",
        onNavClick   = { route -> navController.navigate(route) { launchSingleTop = true } },
        onStoreClick = { navController.navigate(Screen.Home.route) },
        onLogout     = {
            authViewModel.logout()
            navController.navigate(Screen.Login.route) { popUpTo(0) { inclusive = true } }
        },
    ) { padding ->
        Box(modifier = Modifier.padding(padding)) {
            CategoriesAdminScreen()
        }
    }
}
```
ArRchivo completo
presentation/navigation/NavGraph.kt

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
import com.shopapp.presentation.ui.admin.dashboard.DashboardScreen
import com.shopapp.presentation.ui.admin.categories.CategoriesAdminScreen
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

    var showCart by remember { mutableStateOf(false) }
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

            composable(Screen.Home.route) {
                HomeScreen(
                    onProductClick = { id -> navController.navigate("product/$id") },
                    onCatalogClick = { navController.navigate(Screen.Catalog.route) },
                )
            }

            composable(Screen.Catalog.route) {
                CatalogScreen(
                    onProductClick = { id -> navController.navigate("product/$id") },
                )
            }

            composable(
                route = "product/{id}",
                arguments = listOf(navArgument("id") { type = NavType.IntType }),
            ) { backStackEntry ->
                val id = backStackEntry.arguments?.getInt("id") ?: return@composable
                ProductDetailScreen(
                    productId     = id,
                    onBack        = { navController.popBackStack() },
                    cartViewModel = cartViewModel,
                )
            }

            composable(Screen.Orders.route) {
                if (!isAuthenticated) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Login.route) {
                            popUpTo(Screen.Home.route)
                        }
                    }
                } else {
                    OrdersScreen(
                        onOrderClick = { id ->
                            navController.navigate("orders/$id")
                        },
                    )
                }
            }

            composable(
                route = "orders/{id}",
                arguments = listOf(navArgument("id") { type = NavType.IntType }),
            ) { backStackEntry ->
                val id = backStackEntry.arguments?.getInt("id") ?: return@composable
                OrderDetailScreen(
                    orderId = id,
                    onBack  = { navController.popBackStack() },
                )
            }

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
                    onStoreClick = {
                        navController.navigate(Screen.Home.route)
                    },
                    onLogout = {
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

            // 🔥 REEMPLAZADO SOLO ESTE
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

            // placeholders restantes
            listOf(
                "admin/products" to "Productos",
                "admin/orders"   to "Pedidos",
                "admin/users"    to "Usuarios",
            ).forEach { (route, title) ->
                composable(route) {
                    if (!isStaff) {
                        LaunchedEffect(Unit) {
                            navController.navigate(Screen.Home.route) { popUpTo(0) }
                        }
                        return@composable
                    }

                    AdminScaffold(
                        currentRoute = route,
                        user         = currentUser,
                        title        = title,
                        onNavClick   = { r -> navController.navigate(r) { launchSingleTop = true } },
                        onStoreClick = { navController.navigate(Screen.Home.route) },
                        onLogout     = {
                            authViewModel.logout()
                            navController.navigate(Screen.Login.route) {
                                popUpTo(0) { inclusive = true }
                            }
                        },
                    ) { padding ->
                        Box(
                            modifier = Modifier.fillMaxSize().padding(padding),
                            contentAlignment = Alignment.Center,
                        ) {
                            Text("$title — próximo módulo", color = TextSecondary)
                        }
                    }
                }
            }
        }
    }
}

```
---

## ✅ Checkpoint Módulo 8

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Drawer → "Categorías" | Lista de categorías con toggle |
| 2 | Buscar "electro" | Filtrado en tiempo real |
| 3 | Toggle del Switch | Cambia de activa a inactiva al instante |
| 4 | Switch desactivado sin conexión | Revierte al estado original |
| 5 | Botón "Nueva" | Bottom sheet con formulario |
| 6 | Escribir nombre | Slug se auto-genera |
| 7 | Editar slug manualmente | Se fija y no cambia al editar nombre |
| 8 | Crear categoría válida | Aparece al inicio de la lista |
| 9 | Botón ✏️ editar | Sheet con datos precargados |
| 10 | Eliminar con productos | Diálogo: "¿Desactivar?" |
| 11 | Eliminar sin productos | Diálogo: "¿Eliminar permanentemente?" |
| 12 | Verificar en catálogo público | La categoría activa/inactiva refleja el cambio |

---

## Resumen

| Elemento | Estado |
|---|---|
| `CategoriesAdminViewModel` con toggle optimista | ✅ |
| `filtered` StateFlow derivado para búsqueda local | ✅ |
| `CategoryFormSheet` con BottomSheet Material 3 | ✅ |
| Auto-generación de slug desde el nombre | ✅ |
| `Switch` con colores del design system | ✅ |
| `CategoriesAdminScreen` con búsqueda y lista | ✅ |
| `CategoryAdminCard` con toggle, editar, eliminar | ✅ |
| `AlertDialog` inteligente (desactivar vs. eliminar) | ✅ |
| `CategoryFormState` sealed interface | ✅ |
| Ruta `admin/categories` en NavGraph | ✅ |

**Siguiente módulo →** M9: Admin CRUD Productos — tabla, crear, editar, restock e imagen