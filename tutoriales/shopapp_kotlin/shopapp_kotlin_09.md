# ShopApp Android — Módulo 9
## Admin CRUD Productos — Lista, crear, editar, restock e imagen

> **Objetivo:** CRUD completo de productos con filtros por stock/estado, formulario con validación, modal de restock y toggle optimista.
> **Checkpoint final:** crear un producto, hacer restock y verificar el cambio de stock en el catálogo público.

---

## 9.1 ViewModel de productos admin — `presentation/viewmodel/ProductsAdminViewModel.kt`

> **Corrección aplicada:** se añadió `CategoryRepository` al constructor y se expone `val categories: StateFlow<List<Category>>`. Esto elimina la necesidad de pasar `categoryRepository` como parámetro al composable y mantiene el patrón Hilt consistente con el resto del proyecto.

```kotlin
// presentation/viewmodel/ProductsAdminViewModel.kt
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.Category
import com.shopapp.domain.model.Product
import com.shopapp.domain.model.ProductFilters
import com.shopapp.domain.model.ProductPayload
import com.shopapp.domain.repository.CategoryRepository
import com.shopapp.domain.repository.ProductRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

enum class ProductStockFilter(val label: String) {
    ALL("Todos"),
    IN_STOCK("Con stock"),
    OUT_OF_STOCK("Sin stock"),
    ACTIVE("Activos"),
    INACTIVE("Inactivos"),
}

data class ProductsAdminUiState(
    val products:    List<Product>      = emptyList(),
    val isLoading:   Boolean            = false,
    val error:       String?            = null,
    val total:       Int                = 0,
    val search:      String             = "",
    val stockFilter: ProductStockFilter = ProductStockFilter.ALL,
)

sealed interface ProductFormState {
    data object Idle                       : ProductFormState
    data object Saving                     : ProductFormState
    data class  Success(val msg: String)   : ProductFormState
    data class  Error(val message: String) : ProductFormState
}

@HiltViewModel
class ProductsAdminViewModel @Inject constructor(
    private val repository:         ProductRepository,
    private val categoryRepository: CategoryRepository, // ← requerido para exponer categories
) : ViewModel() {

    // ── UI state ──────────────────────────────────────────────────
    private val _state = MutableStateFlow(ProductsAdminUiState())
    val state: StateFlow<ProductsAdminUiState> = _state.asStateFlow()

    private val _formState = MutableStateFlow<ProductFormState>(ProductFormState.Idle)
    val formState: StateFlow<ProductFormState> = _formState.asStateFlow()

    // ── Categorías ────────────────────────────────────────────────
    private val _categories = MutableStateFlow<List<Category>>(emptyList())
    val categories: StateFlow<List<Category>> = _categories.asStateFlow()

    // ── Filtrado local combinado (búsqueda + filtro de stock) ─────
    val filtered: StateFlow<List<Product>> = _state
        .map { s ->
            s.products
                .filter { p ->
                    s.search.isBlank() || p.name.contains(s.search, ignoreCase = true)
                }
                .filter { p ->
                    when (s.stockFilter) {
                        ProductStockFilter.ALL          -> true
                        ProductStockFilter.IN_STOCK     -> p.stock > 0
                        ProductStockFilter.OUT_OF_STOCK -> p.stock == 0
                        ProductStockFilter.ACTIVE       -> p.isActive
                        ProductStockFilter.INACTIVE     -> !p.isActive
                    }
                }
        }
        .stateIn(viewModelScope, SharingStarted.Eagerly, emptyList())

    init {
        load()
        loadCategories()
    }

    // ── Carga de datos ────────────────────────────────────────────

    fun load() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            repository.getProducts(ProductFilters(pageSize = 50))
                .onSuccess { (products, total) ->
                    _state.update { it.copy(products = products, total = total, isLoading = false) }
                }
                .onFailure { e ->
                    _state.update { it.copy(isLoading = false, error = e.message) }
                }
        }
    }

    private fun loadCategories() {
        viewModelScope.launch {
            categoryRepository.getCategories()
                .onSuccess { _categories.value = it }
                // Si falla, la lista queda vacía; el dropdown del formulario aparecerá vacío
        }
    }

    // ── Filtros ───────────────────────────────────────────────────

    fun setSearch(query: String)                   = _state.update { it.copy(search = query) }
    fun setStockFilter(filter: ProductStockFilter) = _state.update { it.copy(stockFilter = filter) }

    // ── Toggle activo (optimista) ─────────────────────────────────

    fun toggleActive(id: Int, isActive: Boolean) {
        _state.update { s ->
            s.copy(products = s.products.map {
                if (it.id == id) it.copy(isActive = isActive) else it
            })
        }
        viewModelScope.launch {
            val product = _state.value.products.firstOrNull { it.id == id } ?: return@launch
            repository.updateProduct(
                id, ProductPayload(
                    name        = product.name,
                    description = product.description,
                    price       = product.price,
                    stock       = product.stock,
                    isActive    = isActive,
                    categoryId  = product.categoryId ?: 0,
                )
            ).onFailure {
                // Revertir si el servidor rechaza el cambio
                _state.update { s ->
                    s.copy(products = s.products.map { p ->
                        if (p.id == id) p.copy(isActive = !isActive) else p
                    })
                }
            }
        }
    }

    // ── CRUD ──────────────────────────────────────────────────────

    fun createProduct(payload: ProductPayload) {
        _formState.value = ProductFormState.Saving
        viewModelScope.launch {
            repository.createProduct(payload)
                .onSuccess { created ->
                    _state.update { s ->
                        s.copy(products = listOf(created) + s.products, total = s.total + 1)
                    }
                    _formState.value = ProductFormState.Success("Producto creado")
                }
                .onFailure { e ->
                    _formState.value = ProductFormState.Error(e.message ?: "Error al crear")
                }
        }
    }

    fun updateProduct(id: Int, payload: ProductPayload) {
        _formState.value = ProductFormState.Saving
        viewModelScope.launch {
            repository.updateProduct(id, payload)
                .onSuccess { updated ->
                    _state.update { s ->
                        s.copy(products = s.products.map { if (it.id == id) updated else it })
                    }
                    _formState.value = ProductFormState.Success("Producto actualizado")
                }
                .onFailure { e ->
                    _formState.value = ProductFormState.Error(e.message ?: "Error al actualizar")
                }
        }
    }

    fun restock(id: Int, quantity: Int, onResult: (String) -> Unit) {
        viewModelScope.launch {
            repository.restock(id, quantity)
                .onSuccess { newStock ->
                    _state.update { s ->
                        s.copy(products = s.products.map { p ->
                            if (p.id == id) p.copy(stock = newStock, inStock = newStock > 0) else p
                        })
                    }
                    onResult("Stock actualizado: $newStock unidades")
                }
                .onFailure { e ->
                    onResult("Error: ${e.message}")
                }
        }
    }

    fun deleteProduct(id: Int) {
        viewModelScope.launch {
            repository.deleteProduct(id)
                .onSuccess {
                    _state.update { s ->
                        s.copy(
                            products = s.products.filter { it.id != id },
                            total    = s.total - 1,
                        )
                    }
                }
                .onFailure { e ->
                    _state.update { it.copy(error = e.message) }
                }
        }
    }

    fun resetFormState() {
        _formState.value = ProductFormState.Idle
    }
}
```

---

## 9.2 Formulario de producto — `presentation/ui/admin/products/ProductFormSheet.kt`

```kotlin
// presentation/ui/admin/products/ProductFormSheet.kt
package com.shopapp.presentation.ui.admin.products

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
import com.shopapp.domain.model.Category
import com.shopapp.domain.model.Product
import com.shopapp.domain.model.ProductPayload
import com.shopapp.presentation.components.ShopTextField
import com.shopapp.presentation.viewmodel.ProductFormState
import com.shopapp.theme.*

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ProductFormSheet(
    initial:    Product?,
    categories: List<Category>,
    formState:  ProductFormState,
    onSave:     (ProductPayload) -> Unit,
    onDismiss:  () -> Unit,
) {
    val isEdit = initial != null

    var name        by remember { mutableStateOf(initial?.name        ?: "") }
    var description by remember { mutableStateOf(initial?.description ?: "") }
    var price       by remember { mutableStateOf(if (initial != null) "%.2f".format(initial.price) else "") }
    var stock       by remember { mutableStateOf(initial?.stock?.toString() ?: "") }
    var isActive    by remember { mutableStateOf(initial?.isActive ?: true) }
    var selectedCat by remember { mutableStateOf(initial?.categoryId) }
    var catExpanded by remember { mutableStateOf(false) }

    val isSaving   = formState is ProductFormState.Saving
    val priceVal   = price.toDoubleOrNull()
    val stockVal   = stock.toIntOrNull()
    val nameError  = name.isNotEmpty() && name.length < 2
    val priceError = price.isNotEmpty() && (priceVal == null || priceVal <= 0)
    val stockError = stock.isNotEmpty() && (stockVal == null || stockVal < 0)
    val canSave    = name.length >= 2 && priceVal != null && priceVal > 0 &&
                     stockVal != null && stockVal >= 0 &&
                     selectedCat != null && !isSaving

    LaunchedEffect(formState) {
        if (formState is ProductFormState.Success) onDismiss()
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
                text       = if (isEdit) "Editar: ${initial?.name}" else "Nuevo producto",
                style      = MaterialTheme.typography.titleLarge,
                fontWeight = FontWeight.Bold,
                color      = TextPrimary,
            )

            // Error global
            if (formState is ProductFormState.Error) {
                Surface(
                    color    = Error.copy(alpha = 0.1f),
                    shape    = MaterialTheme.shapes.small,
                    modifier = Modifier.fillMaxWidth(),
                ) {
                    Text(
                        formState.message, color = Error,
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
                placeholder   = "ej. Laptop HP 15",
                isError       = nameError,
                errorMessage  = "Mínimo 2 caracteres",
                enabled       = !isSaving,
            )

            // Descripción
            OutlinedTextField(
                value         = description,
                onValueChange = { description = it },
                label         = { Text("Descripción") },
                minLines      = 3, maxLines = 4,
                enabled       = !isSaving,
                modifier      = Modifier.fillMaxWidth(),
                colors        = OutlinedTextFieldDefaults.colors(
                    focusedBorderColor   = Accent,
                    unfocusedBorderColor = Border,
                    focusedLabelColor    = Accent,
                    unfocusedLabelColor  = TextSecondary,
                ),
            )

            // Precio y Stock en fila
            Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                ShopTextField(
                    value         = price,
                    onValueChange = { price = it },
                    label         = "Precio $ *",
                    placeholder   = "0.00",
                    isError       = priceError,
                    errorMessage  = "Precio inválido",
                    enabled       = !isSaving,
                    keyboardType  = KeyboardType.Decimal,
                    modifier      = Modifier.weight(1f),
                )
                ShopTextField(
                    value         = stock,
                    onValueChange = { stock = it },
                    label         = "Stock *",
                    placeholder   = "0",
                    isError       = stockError,
                    errorMessage  = "Stock inválido",
                    enabled       = !isSaving,
                    keyboardType  = KeyboardType.Number,
                    modifier      = Modifier.weight(1f),
                )
            }

            // Selector de categoría con ExposedDropdownMenu
            ExposedDropdownMenuBox(
                expanded         = catExpanded,
                onExpandedChange = { catExpanded = !catExpanded },
            ) {
                OutlinedTextField(
                    value         = categories.find { it.id == selectedCat }?.name ?: "— Seleccionar —",
                    onValueChange = {},
                    readOnly      = true,
                    label         = { Text("Categoría *") },
                    trailingIcon  = { ExposedDropdownMenuDefaults.TrailingIcon(expanded = catExpanded) },
                    colors        = OutlinedTextFieldDefaults.colors(
                        focusedBorderColor   = Accent,
                        unfocusedBorderColor = if (selectedCat == null) Error else Border,
                        focusedLabelColor    = Accent,
                        unfocusedLabelColor  = TextSecondary,
                    ),
                    modifier = Modifier
                        .menuAnchor()
                        .fillMaxWidth(),
                )
                ExposedDropdownMenu(
                    expanded         = catExpanded,
                    onDismissRequest = { catExpanded = false },
                ) {
                    categories.filter { it.isActive }.forEach { cat ->
                        DropdownMenuItem(
                            text = {
                                Text(
                                    cat.name,
                                    color      = if (selectedCat == cat.id) Accent else TextPrimary,
                                    fontWeight = if (selectedCat == cat.id) FontWeight.Bold else FontWeight.Normal,
                                )
                            },
                            onClick = { selectedCat = cat.id; catExpanded = false },
                        )
                    }
                }
            }

            // Toggle activo
            Surface(
                color    = Surface2,
                shape    = MaterialTheme.shapes.medium,
                modifier = Modifier.fillMaxWidth(),
            ) {
                Row(
                    modifier              = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp, vertical = 12.dp),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment     = Alignment.CenterVertically,
                ) {
                    Column {
                        Text(
                            "Producto activo",
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
                            checkedThumbColor    = AccentOnDark,
                            checkedTrackColor    = Accent,
                            uncheckedTrackColor  = Surface,
                            uncheckedBorderColor = Border,
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
                    shape = MaterialTheme.shapes.medium,
                ) { Text("Cancelar") }

                Button(
                    onClick = {
                        onSave(ProductPayload(
                            name        = name.trim(),
                            description = description.trim(),
                            price       = priceVal!!,
                            stock       = stockVal!!,
                            isActive    = isActive,
                            categoryId  = selectedCat!!,
                        ))
                    },
                    enabled  = canSave,
                    modifier = Modifier.weight(1f).height(52.dp),
                    colors   = ButtonDefaults.buttonColors(
                        containerColor         = Accent,
                        contentColor           = AccentOnDark,
                        disabledContainerColor = Accent.copy(alpha = 0.4f),
                    ),
                    shape = MaterialTheme.shapes.medium,
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
                        else "Crear producto",
                        fontWeight = FontWeight.Bold,
                    )
                }
            }
        }
    }
}
```

---

## 9.3 Modal de restock — `presentation/ui/admin/products/RestockDialog.kt`

```kotlin
// presentation/ui/admin/products/RestockDialog.kt
package com.shopapp.presentation.ui.admin.products

import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.input.KeyboardType
import androidx.compose.ui.unit.dp
import com.shopapp.domain.model.Product
import com.shopapp.presentation.components.ShopTextField
import com.shopapp.theme.*

@Composable
fun RestockDialog(
    product:   Product,
    onRestock: (Int) -> Unit,
    onDismiss: () -> Unit,
) {
    var qty       by remember { mutableStateOf("") }
    var isLoading by remember { mutableStateOf(false) }
    var feedback  by remember { mutableStateOf<String?>(null) }

    val qtyVal     = qty.toIntOrNull()
    val qtyError   = qty.isNotEmpty() && (qtyVal == null || qtyVal <= 0)
    val canRestock = qtyVal != null && qtyVal > 0 && !isLoading

    AlertDialog(
        onDismissRequest = { if (!isLoading) onDismiss() },
        containerColor   = Surface,
        shape            = MaterialTheme.shapes.large,
        title = {
            Column {
                Text(
                    "Restock: ${product.name}",
                    style      = MaterialTheme.typography.titleMedium,
                    fontWeight = FontWeight.Bold,
                    color      = TextPrimary,
                )
                Spacer(Modifier.height(4.dp))
                Text(
                    "Stock actual: ${product.stock} unidades",
                    style = MaterialTheme.typography.bodySmall,
                    color = if (product.stock == 0) Error else TextSecondary,
                )
            }
        },
        text = {
            Column(verticalArrangement = Arrangement.spacedBy(12.dp)) {
                ShopTextField(
                    value         = qty,
                    onValueChange = { qty = it; feedback = null },
                    label         = "Cantidad a añadir *",
                    placeholder   = "ej. 50",
                    isError       = qtyError,
                    errorMessage  = "Debe ser mayor que 0",
                    enabled       = !isLoading,
                    keyboardType  = KeyboardType.Number,
                )
                if (qtyVal != null && qtyVal > 0) {
                    Text(
                        "Nuevo stock: ${product.stock + qtyVal} unidades",
                        style = MaterialTheme.typography.bodySmall,
                        color = TextSecondary,
                    )
                }
                feedback?.let {
                    Text(
                        it,
                        style      = MaterialTheme.typography.bodySmall,
                        color      = if (it.startsWith("Error")) Error else Success,
                        fontWeight = FontWeight.SemiBold,
                    )
                }
            }
        },
        confirmButton = {
            Button(
                onClick = { isLoading = true; onRestock(qtyVal!!) },
                enabled = canRestock,
                colors  = ButtonDefaults.buttonColors(
                    containerColor         = Accent,
                    contentColor           = AccentOnDark,
                    disabledContainerColor = Accent.copy(alpha = 0.4f),
                ),
            ) {
                if (isLoading) {
                    CircularProgressIndicator(
                        color       = AccentOnDark,
                        modifier    = Modifier.size(14.dp),
                        strokeWidth = 2.dp,
                    )
                    Spacer(Modifier.width(6.dp))
                }
                Text(if (isLoading) "Guardando..." else "Añadir stock", fontWeight = FontWeight.Bold)
            }
        },
        dismissButton = {
            TextButton(onClick = { if (!isLoading) onDismiss() }) {
                Text("Cancelar", color = TextSecondary)
            }
        },
    )
}
```

---

## 9.4 Pantalla de productos admin — `presentation/ui/admin/products/ProductsAdminScreen.kt`

> **Corrección aplicada:** se eliminó el parámetro `categoryRepository: CategoryRepository` de la firma. Las categorías se leen directamente desde `viewModel.categories`, que el ViewModel resuelve internamente vía Hilt.

```kotlin
// presentation/ui/admin/products/ProductsAdminScreen.kt
package com.shopapp.presentation.ui.admin.products

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.hilt.navigation.compose.hiltViewModel
import coil.compose.AsyncImage
import com.shopapp.domain.model.Product
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.components.ErrorScreen
import com.shopapp.presentation.viewmodel.ProductStockFilter
import com.shopapp.presentation.viewmodel.ProductsAdminViewModel
import com.shopapp.theme.*

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ProductsAdminScreen(
    viewModel: ProductsAdminViewModel = hiltViewModel(),
) {
    val state      by viewModel.state.collectAsState()
    val filtered   by viewModel.filtered.collectAsState()
    val formState  by viewModel.formState.collectAsState()
    val categories by viewModel.categories.collectAsState() // ← desde el ViewModel

    var showForm      by remember { mutableStateOf(false) }
    var editTarget    by remember { mutableStateOf<Product?>(null) }
    var restockTarget by remember { mutableStateOf<Product?>(null) }
    var deleteTarget  by remember { mutableStateOf<Product?>(null) }
    var snackMsg      by remember { mutableStateOf<String?>(null) }

    val snackbarHostState = remember { SnackbarHostState() }

    LaunchedEffect(snackMsg) {
        snackMsg?.let {
            snackbarHostState.showSnackbar(it)
            snackMsg = null
        }
    }

    Scaffold(
        snackbarHost   = { SnackbarHost(snackbarHostState) },
        containerColor = Background,
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
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
                                "Productos",
                                style      = MaterialTheme.typography.headlineMedium,
                                fontWeight = FontWeight.Bold,
                                color      = TextPrimary,
                            )
                            Text(
                                "${state.total} productos",
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
                                    containerColor = Accent,
                                    contentColor   = AccentOnDark,
                                ),
                                shape          = MaterialTheme.shapes.medium,
                                contentPadding = PaddingValues(horizontal = 16.dp, vertical = 10.dp),
                            ) {
                                Icon(Icons.Default.Add, null, modifier = Modifier.size(18.dp))
                                Spacer(Modifier.width(4.dp))
                                Text("Nuevo", fontWeight = FontWeight.Bold)
                            }
                        }
                    }

                    Spacer(Modifier.height(12.dp))

                    OutlinedTextField(
                        value         = state.search,
                        onValueChange = viewModel::setSearch,
                        placeholder   = { Text("Buscar producto...", color = TextFaint) },
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

                    LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        items(ProductStockFilter.entries) { filter ->
                            FilterChip(
                                selected = state.stockFilter == filter,
                                onClick  = { viewModel.setStockFilter(filter) },
                                label    = {
                                    Text(filter.label, style = MaterialTheme.typography.labelSmall)
                                },
                                colors = FilterChipDefaults.filterChipColors(
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

            // ── Lista ──────────────────────────────────────────────
            when {
                state.isLoading     -> LoadingScreen("Cargando productos...")
                state.error != null -> ErrorScreen(state.error!!, onRetry = viewModel::load)
                filtered.isEmpty()  -> {
                    Box(Modifier.fillMaxSize(), Alignment.Center) {
                        Column(horizontalAlignment = Alignment.CenterHorizontally) {
                            Text("📦", fontSize = 48.sp)
                            Spacer(Modifier.height(12.dp))
                            Text(
                                if (state.search.isBlank() && state.stockFilter == ProductStockFilter.ALL)
                                    "Sin productos"
                                else
                                    "Sin resultados",
                                style      = MaterialTheme.typography.titleMedium,
                                fontWeight = FontWeight.Bold,
                                color      = TextPrimary,
                            )
                        }
                    }
                }
                else -> {
                    LazyColumn(
                        modifier            = Modifier.fillMaxSize(),
                        contentPadding      = PaddingValues(16.dp),
                        verticalArrangement = Arrangement.spacedBy(10.dp),
                    ) {
                        items(filtered, key = { it.id }) { product ->
                            ProductAdminCard(
                                product   = product,
                                onToggle  = { viewModel.toggleActive(product.id, !product.isActive) },
                                onEdit    = { editTarget = product; showForm = true },
                                onRestock = { restockTarget = product },
                                onDelete  = { deleteTarget = product },
                            )
                        }
                    }
                }
            }
        }
    }

    // ── Bottom Sheet: formulario crear / editar ────────────────────
    if (showForm) {
        ProductFormSheet(
            initial    = editTarget,
            categories = categories,
            formState  = formState,
            onSave     = { payload ->
                if (editTarget != null) viewModel.updateProduct(editTarget!!.id, payload)
                else viewModel.createProduct(payload)
            },
            onDismiss = {
                showForm   = false
                editTarget = null
                viewModel.resetFormState()
            },
        )
    }

    // ── Dialog: restock ───────────────────────────────────────────
    restockTarget?.let { product ->
        RestockDialog(
            product   = product,
            onRestock = { qty ->
                viewModel.restock(product.id, qty) { msg ->
                    snackMsg      = msg
                    restockTarget = null
                }
            },
            onDismiss = { restockTarget = null },
        )
    }

    // ── Dialog: confirmación de eliminación ───────────────────────
    deleteTarget?.let { product ->
        AlertDialog(
            onDismissRequest = { deleteTarget = null },
            containerColor   = Surface,
            shape            = MaterialTheme.shapes.large,
            title            = { Text("¿Eliminar producto?", color = TextPrimary) },
            text             = {
                Text(
                    "\"${product.name}\" se eliminará permanentemente.",
                    color = TextSecondary,
                )
            },
            confirmButton = {
                TextButton(onClick = {
                    viewModel.deleteProduct(product.id)
                    deleteTarget = null
                }) {
                    Text("Eliminar", color = Error, fontWeight = FontWeight.Bold)
                }
            },
            dismissButton = {
                TextButton(onClick = { deleteTarget = null }) {
                    Text("Cancelar", color = TextSecondary)
                }
            },
        )
    }
}

// ── ProductAdminCard ──────────────────────────────────────────────────────────

@Composable
private fun ProductAdminCard(
    product:   Product,
    onToggle:  () -> Unit,
    onEdit:    () -> Unit,
    onRestock: () -> Unit,
    onDelete:  () -> Unit,
) {
    Surface(
        shape    = MaterialTheme.shapes.large,
        color    = if (product.isActive) Surface else Surface.copy(alpha = 0.6f),
        modifier = Modifier.fillMaxWidth(),
    ) {
        Row(
            modifier          = Modifier.padding(12.dp),
            verticalAlignment = Alignment.CenterVertically,
        ) {
            // Imagen
            Box(
                modifier         = Modifier
                    .size(56.dp)
                    .background(Surface2, MaterialTheme.shapes.medium),
                contentAlignment = Alignment.Center,
            ) {
                if (product.imageUrl != null) {
                    AsyncImage(
                        model              = product.imageUrl,
                        contentDescription = product.name,
                        contentScale       = ContentScale.Crop,
                        modifier           = Modifier.fillMaxSize(),
                    )
                } else {
                    Text("📦", fontSize = 22.sp)
                }
            }

            Spacer(Modifier.width(12.dp))

            // Info
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text       = product.name,
                    style      = MaterialTheme.typography.bodyMedium,
                    fontWeight = FontWeight.SemiBold,
                    color      = TextPrimary,
                    maxLines   = 1,
                    overflow   = TextOverflow.Ellipsis,
                )
                Text(
                    text  = product.categoryName ?: "Sin categoría",
                    style = MaterialTheme.typography.bodySmall,
                    color = TextSecondary,
                )
                Row(
                    horizontalArrangement = Arrangement.spacedBy(8.dp),
                    verticalAlignment     = Alignment.CenterVertically,
                ) {
                    Text(
                        "$${"%.2f".format(product.price)}",
                        style      = MaterialTheme.typography.bodyMedium,
                        fontWeight = FontWeight.Bold,
                        color      = Accent,
                    )
                    // Badge de stock
                    Surface(
                        color = when {
                            product.stock == 0 -> Error.copy(alpha = 0.15f)
                            product.stock < 5  -> Warning.copy(alpha = 0.15f)
                            else               -> Success.copy(alpha = 0.12f)
                        },
                        shape = MaterialTheme.shapes.extraSmall,
                    ) {
                        Text(
                            text = when {
                                product.stock == 0 -> "Agotado"
                                else               -> "${product.stock} uds."
                            },
                            color = when {
                                product.stock == 0 -> Error
                                product.stock < 5  -> Warning
                                else               -> Success
                            },
                            fontSize   = 10.sp,
                            fontWeight = FontWeight.Bold,
                            modifier   = Modifier.padding(horizontal = 6.dp, vertical = 2.dp),
                        )
                    }
                }
            }

            // Acciones
            Column(horizontalAlignment = Alignment.End) {
                Switch(
                    checked         = product.isActive,
                    onCheckedChange = { onToggle() },
                    colors          = SwitchDefaults.colors(
                        checkedThumbColor    = AccentOnDark,
                        checkedTrackColor    = Accent,
                        uncheckedTrackColor  = Surface2,
                        uncheckedBorderColor = Border,
                    ),
                    modifier = Modifier.size(width = 48.dp, height = 28.dp),
                )
                Spacer(Modifier.height(4.dp))
                Row {
                    IconButton(onClick = onRestock, modifier = Modifier.size(32.dp)) {
                        Icon(
                            Icons.Default.Inventory, null,
                            tint     = Accent,
                            modifier = Modifier.size(18.dp),
                        )
                    }
                    IconButton(onClick = onEdit, modifier = Modifier.size(32.dp)) {
                        Icon(
                            Icons.Default.Edit, null,
                            tint     = TextSecondary,
                            modifier = Modifier.size(18.dp),
                        )
                    }
                    IconButton(onClick = onDelete, modifier = Modifier.size(32.dp)) {
                        Icon(
                            Icons.Default.Delete, null,
                            tint     = Error,
                            modifier = Modifier.size(18.dp),
                        )
                    }
                }
            }
        }
    }
}
```

---

## 9.5 Actualizar NavGraph — `presentation/navigation/NavGraph.kt`

> **Corrección aplicada:** la ruta `admin/products` llama a `ProductsAdminScreen()` sin parámetros. No se necesita el wrapper intermedio ni pasar `categoryRepository` manualmente — Hilt resuelve todo a través del ViewModel.

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
import com.shopapp.presentation.ui.admin.products.ProductsAdminScreen
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

        // ── BottomSheet del carrito ───────────────────────────
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

            // ── LOGIN ───────────────────────────────────────────
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

            // ── REGISTER ────────────────────────────────────────
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

            // ── HOME ────────────────────────────────────────────
            composable(Screen.Home.route) {
                HomeScreen(
                    onProductClick = { id -> navController.navigate("product/$id") },
                    onCatalogClick = { navController.navigate(Screen.Catalog.route) },
                )
            }

            // ── CATÁLOGO ────────────────────────────────────────
            composable(Screen.Catalog.route) {
                CatalogScreen(
                    onProductClick = { id -> navController.navigate("product/$id") },
                )
            }

            // ── DETALLE PRODUCTO ────────────────────────────────
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

            // ── ORDERS ──────────────────────────────────────────
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

            // ── DETALLE PEDIDO ──────────────────────────────────
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

            // ── PROFILE ─────────────────────────────────────────
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

            // ── ADMIN DASHBOARD ─────────────────────────────────
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

            // ── ADMIN CATEGORÍAS ────────────────────────────────
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

            // ── ADMIN PRODUCTOS ─────────────────────────────────
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

            // ── PLACEHOLDERS (M10+) ─────────────────────────────
            listOf(
                "admin/orders" to "Pedidos",
                "admin/users"  to "Usuarios",
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
                            modifier         = Modifier
                                .fillMaxSize()
                                .padding(padding),
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

## ✅ Checkpoint Módulo 9

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Drawer → "Productos" | Lista con imagen, precio y badge de stock |
| 2 | Filtro "Sin stock" | Solo productos agotados |
| 3 | Buscar por nombre | Filtrado local instantáneo |
| 4 | Toggle activo | Cambia al instante (optimista) |
| 5 | Ícono 📦 Restock | Dialog con stock actual |
| 6 | Restock con 50 | Snackbar "Stock actualizado: N unidades" |
| 7 | Botón "Nuevo" | Bottom sheet con formulario |
| 8 | Crear sin categoría | Botón deshabilitado |
| 9 | Precio negativo | Error "Precio inválido" |
| 10 | Crear producto válido | Aparece en la lista |
| 11 | Editar precio | Precio actualizado en la lista |
| 12 | Verificar en catálogo | `/catalog` muestra el nuevo producto |

---

## Resumen

| Elemento | Estado |
|----------|--------|
| `ProductsAdminViewModel` — `CategoryRepository` inyectado, `categories` expuesto | ✅ |
| `ProductsAdminViewModel` — toggle optimista y restock | ✅ |
| `filtered` con búsqueda + filtro de stock combinados | ✅ |
| `ProductFormSheet` con `ExposedDropdownMenu` | ✅ |
| `RestockDialog` con preview del nuevo stock | ✅ |
| `ProductsAdminScreen` — sin parámetro `categoryRepository` | ✅ |
| `ProductsAdminScreen` — `Scaffold` + `SnackbarHost` | ✅ |
| `ProductAdminCard` — imagen, badge de stock, 3 acciones | ✅ |
| `NavGraph` — `admin/products` llama `ProductsAdminScreen()` sin parámetros | ✅ |
| `ProductStockFilter` enum con 5 opciones | ✅ |

**Siguiente módulo →** M10: Admin CRUD Pedidos — lista con filtros, cambio de estado inline y detalle