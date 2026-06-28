# ShopApp Android — Módulo 4
## Navegación completa — NavGraph con guards, Scaffold y Tienda pública

> **Objetivo:** NavGraph definitivo con zonas pública, privada y admin, BottomNavBar para el cliente, TopAppBar contextual, pantalla Home y Catálogo con filtros.
> **Checkpoint final:** navegar por Home y Catálogo, filtrar por categoría, ver el badge del carrito en la barra inferior.

---

## 4.1 Repositorios restantes — `di/RepositoryModule.kt` completo

Antes de seguir, agregar los repositorios de productos y pedidos.

### `data/repository/ProductRepositoryImpl.kt`

```kotlin
// data/repository/ProductRepositoryImpl.kt
package com.shopapp.data.repository

import com.shopapp.data.remote.api.ProductApi
import com.shopapp.data.remote.dto.RestockRequestDto
import com.shopapp.data.remote.dto.toDomain
import com.shopapp.data.remote.dto.toRequest
import com.shopapp.domain.model.Product
import com.shopapp.domain.model.ProductFilters
import com.shopapp.domain.model.ProductPayload
import com.shopapp.domain.repository.ProductRepository
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class ProductRepositoryImpl @Inject constructor(
    private val api: ProductApi,
) : ProductRepository {

    override suspend fun getProducts(filters: ProductFilters): Result<Pair<List<Product>, Int>> =
        runCatching {
            val params = buildMap<String, String> {
                filters.search?.let   { put("search",    it) }
                filters.category?.let { put("category",  it.toString()) }
                filters.priceMin?.let { put("price_min", it.toString()) }
                filters.priceMax?.let { put("price_max", it.toString()) }
                filters.stockMin?.let { put("stock_min", it.toString()) }
                filters.isActive?.let { put("is_active", it.toString()) }
                filters.ordering?.let { put("ordering",  it) }
                put("page",      filters.page.toString())
                put("page_size", filters.pageSize.toString())
            }
            val response = api.getProducts(params)
            if (response.isSuccessful) {
                val body = response.body()!!
                Pair(body.results.map { it.toDomain() }, body.count)
            } else error("Error ${response.code()}")
        }

    override suspend fun getProduct(id: Int): Result<Product> = runCatching {
        val response = api.getProduct(id)
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}")
    }

    override suspend fun createProduct(payload: ProductPayload): Result<Product> = runCatching {
        val response = api.createProduct(payload.toRequest())
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}: ${response.errorBody()?.string()}")
    }

    override suspend fun updateProduct(id: Int, payload: ProductPayload): Result<Product> =
        runCatching {
            val response = api.updateProduct(id, payload.toRequest())
            if (response.isSuccessful) response.body()!!.toDomain()
            else error("Error ${response.code()}: ${response.errorBody()?.string()}")
        }

    override suspend fun deleteProduct(id: Int): Result<Unit> = runCatching {
        val response = api.deleteProduct(id)
        if (!response.isSuccessful) error("Error ${response.code()}")
    }

    override suspend fun restock(id: Int, quantity: Int): Result<Int> = runCatching {
        val response = api.restock(id, RestockRequestDto(quantity))
        if (response.isSuccessful) response.body()!!.newStock
        else error("Error ${response.code()}")
    }

    override suspend fun getStats(): Result<Map<String, Any>> = runCatching {
        val response = api.getStats()
        if (response.isSuccessful) {
            val s = response.body()!!
            mapOf(
                "total_active"   to s.totalActive,
                "total_inactive" to s.totalInactive,
                "avg_price"      to (s.avgPrice ?: 0.0),
                "total_stock"    to (s.totalStock ?: 0),
                "out_of_stock"   to s.outOfStock,
            )
        } else error("Error ${response.code()}")
    }
}
```

### `di/RepositoryModule.kt` — completo

```kotlin
// di/RepositoryModule.kt
package com.shopapp.di

import com.shopapp.data.repository.*
import com.shopapp.domain.repository.*
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

    @Binds @Singleton
    abstract fun bindProductRepository(impl: ProductRepositoryImpl): ProductRepository
}
```

---

## 4.2 CartViewModel — `presentation/viewmodel/CartViewModel.kt`

```kotlin
// presentation/viewmodel/CartViewModel.kt
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import com.shopapp.domain.model.Product
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import javax.inject.Inject

data class CartItem(
    val product:  Product,
    val quantity: Int,
)

@HiltViewModel
class CartViewModel @Inject constructor() : ViewModel() {

    private val _items = MutableStateFlow<List<CartItem>>(emptyList())
    val items: StateFlow<List<CartItem>> = _items.asStateFlow()

    val totalItems: StateFlow<Int> = _items
        .map { list -> list.sumOf { it.quantity } }
        .stateIn(viewModelScope, SharingStarted.Eagerly, 0)

    val subtotal: StateFlow<Double> = _items
        .map { list -> list.sumOf { it.product.price * it.quantity } }
        .stateIn(viewModelScope, SharingStarted.Eagerly, 0.0)

    val totalWithTax: StateFlow<Double> = _items
        .map { list -> list.sumOf { it.product.priceWithTax * it.quantity } }
        .stateIn(viewModelScope, SharingStarted.Eagerly, 0.0)

    fun addItem(product: Product, quantity: Int = 1) {
        _items.update { list ->
            val existing = list.find { it.product.id == product.id }
            if (existing != null) {
                list.map {
                    if (it.product.id == product.id)
                        it.copy(quantity = minOf(it.quantity + quantity, product.stock))
                    else it
                }
            } else {
                list + CartItem(product, quantity)
            }
        }
    }

    fun updateQuantity(productId: Int, quantity: Int) {
        if (quantity <= 0) removeItem(productId)
        else _items.update { list ->
            list.map { if (it.product.id == productId) it.copy(quantity = quantity) else it }
        }
    }

    fun removeItem(productId: Int) {
        _items.update { list -> list.filter { it.product.id != productId } }
    }

    fun clearCart() { _items.value = emptyList() }
}
```

---

## 4.3 BottomNavBar — `presentation/navigation/BottomNavBar.kt`

```kotlin
// presentation/navigation/BottomNavBar.kt
package com.shopapp.presentation.navigation

import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material.icons.outlined.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.unit.dp
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.navigation.NavController
import androidx.navigation.compose.currentBackStackEntryAsState
import com.shopapp.theme.*

data class BottomNavItem(
    val screen:      Screen,
    val label:       String,
    val icon:        ImageVector,
    val iconSelected:ImageVector,
    val badgeCount:  Int = 0,
)

@Composable
fun BottomNavBar(
    navController: NavController,
    cartCount:     Int,
    onCartClick:   () -> Unit,
) {
    val items = listOf(
        BottomNavItem(Screen.Home,   "Inicio",   Icons.Outlined.Home,          Icons.Filled.Home),
        BottomNavItem(Screen.Catalog,"Catálogo", Icons.Outlined.GridView,       Icons.Filled.GridView),
        BottomNavItem(Screen.Cart,   "Carrito",  Icons.Outlined.ShoppingCart,   Icons.Filled.ShoppingCart, cartCount),
        BottomNavItem(Screen.Orders, "Pedidos",  Icons.Outlined.ReceiptLong,    Icons.Filled.ReceiptLong),
        BottomNavItem(Screen.Profile,"Perfil",   Icons.Outlined.AccountCircle,  Icons.Filled.AccountCircle),
    )

    val navBackStackEntry by navController.currentBackStackEntryAsState()
    val currentRoute      = navBackStackEntry?.destination?.route

    NavigationBar(
        containerColor = Surface,
        tonalElevation = 0.dp,
    ) {
        items.forEach { item ->
            val isSelected = currentRoute == item.screen.route

            NavigationBarItem(
                selected = isSelected,
                onClick  = {
                    if (item.screen == Screen.Cart) {
                        onCartClick()
                    } else {
                        navController.navigate(item.screen.route) {
                            popUpTo(Screen.Home.route) { saveState = true }
                            launchSingleTop = true
                            restoreState    = true
                        }
                    }
                },
                icon = {
                    if (item.badgeCount > 0) {
                        BadgedBox(badge = {
                            Badge(containerColor = Error) {
                                Text(
                                    text  = if (item.badgeCount > 99) "99+" else item.badgeCount.toString(),
                                    style = MaterialTheme.typography.labelSmall,
                                )
                            }
                        }) {
                            Icon(
                                imageVector = if (isSelected) item.iconSelected else item.icon,
                                contentDescription = item.label,
                            )
                        }
                    } else {
                        Icon(
                            imageVector = if (isSelected) item.iconSelected else item.icon,
                            contentDescription = item.label,
                        )
                    }
                },
                label = { Text(item.label, style = MaterialTheme.typography.labelSmall) },
                colors = NavigationBarItemDefaults.colors(
                    selectedIconColor       = Accent,
                    selectedTextColor       = Accent,
                    indicatorColor          = Accent.copy(alpha = 0.12f),
                    unselectedIconColor     = TextSecondary,
                    unselectedTextColor     = TextSecondary,
                ),
            )
        }
    }
}
```

---

## 4.4 ViewModel de catálogo — `presentation/viewmodel/CatalogViewModel.kt`

```kotlin
// presentation/viewmodel/CatalogViewModel.kt
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.Category
import com.shopapp.domain.model.Product
import com.shopapp.domain.model.ProductFilters
import com.shopapp.domain.repository.CategoryRepository
import com.shopapp.domain.repository.ProductRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import javax.inject.Inject

data class CatalogUiState(
    val products:         List<Product> = emptyList(),
    val categories:       List<Category> = emptyList(),
    val isLoading:        Boolean = false,
    val isLoadingMore:    Boolean = false,
    val error:            String? = null,
    val total:            Int     = 0,
    val hasMore:          Boolean = false,
    val search:           String  = "",
    val selectedCategory: Int?    = null,
    val ordering:         String  = "",
    val page:             Int     = 1,
)

@HiltViewModel
class CatalogViewModel @Inject constructor(
    private val productRepository:  ProductRepository,
    private val categoryRepository: CategoryRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(CatalogUiState())
    val state: StateFlow<CatalogUiState> = _state.asStateFlow()

    private var searchJob: Job? = null

    init { loadCategories(); load() }

    private fun loadCategories() {
        viewModelScope.launch {
            categoryRepository.getCategories().onSuccess { cats ->
                _state.update { it.copy(categories = cats.filter { c -> c.isActive }) }
            }
        }
    }

    fun load(reset: Boolean = true) {
        val current = _state.value
        val page    = if (reset) 1 else current.page

        if (reset) {
            _state.update { it.copy(isLoading = true, error = null, page = 1) }
        } else {
            if (current.isLoadingMore || !current.hasMore) return
            _state.update { it.copy(isLoadingMore = true) }
        }

        viewModelScope.launch {
            val filters = ProductFilters(
                search   = current.search.ifBlank { null },
                category = current.selectedCategory,
                ordering = current.ordering.ifBlank { null },
                isActive = true,
                page     = page,
                pageSize = 12,
            )
            productRepository.getProducts(filters)
                .onSuccess { (products, total) ->
                    _state.update { s ->
                        s.copy(
                            products      = if (reset) products else s.products + products,
                            total         = total,
                            hasMore       = (if (reset) products else s.products + products).size < total,
                            isLoading     = false,
                            isLoadingMore = false,
                            page          = page + 1,
                            error         = null,
                        )
                    }
                }
                .onFailure { e ->
                    _state.update { it.copy(isLoading = false, isLoadingMore = false, error = e.message) }
                }
        }
    }

    fun setSearch(query: String) {
        _state.update { it.copy(search = query) }
        searchJob?.cancel()
        searchJob = viewModelScope.launch {
            delay(400)
            load(reset = true)
        }
    }

    fun setCategory(id: Int?) {
        _state.update { it.copy(selectedCategory = id) }
        load(reset = true)
    }

    fun setOrdering(ordering: String) {
        _state.update { it.copy(ordering = ordering) }
        load(reset = true)
    }

    fun loadMore() = load(reset = false)
    fun refresh()  = load(reset = true)
}
```

---

## 4.5 Pantalla Home — `presentation/ui/uipublic/home/HomeScreen.kt`

```kotlin
// presentation/ui/uipublic/home/HomeScreen.kt
package com.shopapp.presentation.ui.uipublic.home

import androidx.compose.foundation.background
import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.ArrowForward
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.hilt.navigation.compose.hiltViewModel
import coil.compose.AsyncImage
import com.shopapp.domain.model.Product
import com.shopapp.presentation.viewmodel.CatalogViewModel
import com.shopapp.theme.*

@Composable
fun HomeScreen(
    onProductClick:  (Int) -> Unit,
    onCatalogClick:  () -> Unit,
    viewModel:       CatalogViewModel = hiltViewModel(),
) {
    val state by viewModel.state.collectAsState()

    LazyColumn(
        modifier            = Modifier
            .fillMaxSize()
            .background(Background),
        contentPadding      = PaddingValues(bottom = 24.dp),
    ) {
        // ── Hero ──────────────────────────────────────────────
        item {
            Box(
                modifier = Modifier
                    .fillMaxWidth()
                    .background(
                        brush = androidx.compose.ui.graphics.Brush.verticalGradient(
                            listOf(Surface2, Background),
                        ),
                    )
                    .padding(horizontal = 24.dp, vertical = 48.dp),
            ) {
                Column {
                    Text(
                        text       = "Descubre lo",
                        fontSize   = 32.sp,
                        fontWeight = FontWeight.Normal,
                        color      = TextSecondary,
                    )
                    Text(
                        text       = "extraordinario",
                        fontSize   = 34.sp,
                        fontWeight = FontWeight.Bold,
                        color      = Accent,
                    )
                    Spacer(Modifier.height(16.dp))
                    Text(
                        text  = "Los mejores productos seleccionados para ti.",
                        color = TextSecondary,
                        style = MaterialTheme.typography.bodyMedium,
                    )
                    Spacer(Modifier.height(24.dp))
                    Button(
                        onClick = onCatalogClick,
                        colors  = ButtonDefaults.buttonColors(
                            containerColor = Accent,
                            contentColor   = AccentOnDark,
                        ),
                        shape = MaterialTheme.shapes.medium,
                    ) {
                        Text("Ver catálogo", fontWeight = FontWeight.Bold)
                        Spacer(Modifier.width(8.dp))
                        Icon(Icons.Default.ArrowForward, contentDescription = null, modifier = Modifier.size(16.dp))
                    }
                }
            }
        }

        // ── Categorías ────────────────────────────────────────
        if (state.categories.isNotEmpty()) {
            item {
                SectionHeader(title = "Categorías", onSeeAll = onCatalogClick)
            }
            item {
                LazyRow(
                    contentPadding      = PaddingValues(horizontal = 24.dp),
                    horizontalArrangement = Arrangement.spacedBy(12.dp),
                ) {
                    items(state.categories.take(6)) { cat ->
                        CategoryChip(
                            name     = cat.name,
                            count    = cat.totalProducts,
                            onClick  = { onCatalogClick() },
                        )
                    }
                }
                Spacer(Modifier.height(24.dp))
            }
        }

        // ── Novedades ─────────────────────────────────────────
        item {
            SectionHeader(title = "Novedades", onSeeAll = onCatalogClick)
        }

        if (state.isLoading) {
            item {
                Box(Modifier.fillMaxWidth().height(200.dp), Alignment.Center) {
                    CircularProgressIndicator(color = Accent)
                }
            }
        } else {
            // Grid 2 columnas con LazyVerticalGrid simulado con filas de LazyColumn
            val chunked = state.products.take(4).chunked(2)
            items(chunked) { row ->
                Row(
                    modifier              = Modifier
                        .fillMaxWidth()
                        .padding(horizontal = 16.dp),
                    horizontalArrangement = Arrangement.spacedBy(12.dp),
                ) {
                    row.forEach { product ->
                        ProductCard(
                            product  = product,
                            onClick  = { onProductClick(product.id) },
                            modifier = Modifier.weight(1f),
                        )
                    }
                    // Celda vacía si el row tiene solo 1 elemento
                    if (row.size == 1) Spacer(Modifier.weight(1f))
                }
                Spacer(Modifier.height(12.dp))
            }
        }
    }
}

// ── Composables locales ───────────────────────────────────────

@Composable
private fun SectionHeader(title: String, onSeeAll: () -> Unit) {
    Row(
        modifier              = Modifier
            .fillMaxWidth()
            .padding(horizontal = 24.dp, vertical = 16.dp),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment     = Alignment.CenterVertically,
    ) {
        Text(
            text       = title,
            style      = MaterialTheme.typography.titleLarge,
            fontWeight = FontWeight.Bold,
            color      = TextPrimary,
        )
        TextButton(onClick = onSeeAll) {
            Text("Ver todos", color = Accent, style = MaterialTheme.typography.bodySmall)
        }
    }
}

@Composable
private fun CategoryChip(name: String, count: Int, onClick: () -> Unit) {
    Surface(
        onClick        = onClick,
        shape          = MaterialTheme.shapes.medium,
        color          = Surface2,
        tonalElevation = 0.dp,
        modifier       = Modifier.width(120.dp),
    ) {
        Column(
            modifier            = Modifier.padding(16.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
        ) {
            Text("🏷️", fontSize = 28.sp)
            Spacer(Modifier.height(6.dp))
            Text(
                text       = name,
                style      = MaterialTheme.typography.bodySmall,
                fontWeight = FontWeight.SemiBold,
                color      = TextPrimary,
            )
            Text(
                text  = "$count productos",
                style = MaterialTheme.typography.bodySmall.copy(fontSize = 10.sp),
                color = TextSecondary,
            )
        }
    }
}

@Composable
fun ProductCard(
    product:  Product,
    onClick:  () -> Unit,
    modifier: Modifier = Modifier,
) {
    Surface(
        onClick        = onClick,
        shape          = MaterialTheme.shapes.large,
        color          = Surface,
        tonalElevation = 0.dp,
        modifier       = modifier,
    ) {
        Column {
            // Imagen
            Box(
                modifier          = Modifier
                    .fillMaxWidth()
                    .height(140.dp)
                    .background(Surface2),
                contentAlignment  = Alignment.Center,
            ) {
                if (product.imageUrl != null) {
                    AsyncImage(
                        model             = product.imageUrl,
                        contentDescription = product.name,
                        contentScale      = ContentScale.Crop,
                        modifier          = Modifier.fillMaxSize(),
                    )
                } else {
                    Text("📦", fontSize = 36.sp)
                }
                if (!product.inStock) {
                    Box(
                        modifier         = Modifier
                            .fillMaxWidth()
                            .align(Alignment.BottomCenter)
                            .background(Error.copy(alpha = 0.85f))
                            .padding(4.dp),
                        contentAlignment = Alignment.Center,
                    ) {
                        Text(
                            text  = "Sin stock",
                            color = MaterialTheme.colorScheme.onError,
                            style = MaterialTheme.typography.labelSmall,
                        )
                    }
                }
            }

            // Info
            Column(modifier = Modifier.padding(12.dp)) {
                Text(
                    text     = product.categoryName ?: "Sin categoría",
                    style    = MaterialTheme.typography.labelSmall,
                    color    = Accent,
                    maxLines = 1,
                )
                Spacer(Modifier.height(2.dp))
                Text(
                    text       = product.name,
                    style      = MaterialTheme.typography.bodyMedium,
                    fontWeight = FontWeight.SemiBold,
                    color      = TextPrimary,
                    maxLines   = 2,
                )
                Spacer(Modifier.height(6.dp))
                Text(
                    text       = "$${"%.2f".format(product.price)}",
                    style      = MaterialTheme.typography.titleMedium,
                    fontWeight = FontWeight.Bold,
                    color      = Accent,
                )
                Text(
                    text  = "$${"%.2f".format(product.priceWithTax)} con IVA",
                    style = MaterialTheme.typography.bodySmall,
                    color = TextSecondary,
                )
            }
        }
    }
}
```

---

## 4.6 Pantalla Catálogo — `presentation/ui/uipublic/catalog/CatalogScreen.kt`

```kotlin
// presentation/ui/uipublic/catalog/CatalogScreen.kt
package com.shopapp.presentation.ui.uipublic.catalog

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.grid.*
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Search
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.presentation.ui.uipublic.home.ProductCard
import com.shopapp.presentation.viewmodel.CatalogViewModel
import com.shopapp.theme.*

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CatalogScreen(
    onProductClick: (Int) -> Unit,
    viewModel: CatalogViewModel = hiltViewModel(),
) {
    val state    by viewModel.state.collectAsState()
    val gridState = rememberLazyGridState()

    // Detectar cuando llegar al final para cargar más
    val shouldLoadMore by remember {
        derivedStateOf {
            val lastVisible = gridState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
            val total       = gridState.layoutInfo.totalItemsCount
            lastVisible >= total - 3 && !state.isLoadingMore && state.hasMore
        }
    }

    LaunchedEffect(shouldLoadMore) {
        if (shouldLoadMore) viewModel.loadMore()
    }

    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(Background),
    ) {
        // ── Barra de búsqueda ──────────────────────────────────
        Surface(color = Surface, tonalElevation = 0.dp) {
            Column(modifier = Modifier.padding(16.dp)) {

                // Título y contador
                Row(
                    modifier              = Modifier.fillMaxWidth(),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment     = Alignment.CenterVertically,
                ) {
                    Text(
                        text       = "Catálogo",
                        style      = MaterialTheme.typography.headlineMedium,
                        fontWeight = FontWeight.Bold,
                        color      = TextPrimary,
                    )
                    Text(
                        text  = "${state.total} productos",
                        style = MaterialTheme.typography.bodySmall,
                        color = TextSecondary,
                    )
                }

                Spacer(Modifier.height(12.dp))

                // Campo de búsqueda
                OutlinedTextField(
                    value         = state.search,
                    onValueChange = viewModel::setSearch,
                    placeholder   = { Text("Buscar productos...", color = TextFaint) },
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

                Spacer(Modifier.height(12.dp))

                // Chips de ordenamiento
                LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                    items(listOf(
                        "" to "Relevancia",
                        "price"  to "Precio ↑",
                        "-price" to "Precio ↓",
                        "-created_at" to "Recientes",
                    )) { (value, label) ->
                        FilterChip(
                            selected = state.ordering == value,
                            onClick  = { viewModel.setOrdering(value) },
                            label    = { Text(label, style = MaterialTheme.typography.labelSmall) },
                            colors   = FilterChipDefaults.filterChipColors(
                                selectedContainerColor    = Accent,
                                selectedLabelColor        = AccentOnDark,
                                containerColor            = Surface2,
                                labelColor                = TextSecondary,
                            ),
                        )
                    }
                }

                // Chips de categorías
                if (state.categories.isNotEmpty()) {
                    Spacer(Modifier.height(8.dp))
                    LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        item {
                            FilterChip(
                                selected = state.selectedCategory == null,
                                onClick  = { viewModel.setCategory(null) },
                                label    = { Text("Todas", style = MaterialTheme.typography.labelSmall) },
                                colors   = FilterChipDefaults.filterChipColors(
                                    selectedContainerColor = Accent,
                                    selectedLabelColor     = AccentOnDark,
                                    containerColor         = Surface2,
                                    labelColor             = TextSecondary,
                                ),
                            )
                        }
                        items(state.categories) { cat ->
                            FilterChip(
                                selected = state.selectedCategory == cat.id,
                                onClick  = { viewModel.setCategory(cat.id) },
                                label    = { Text(cat.name, style = MaterialTheme.typography.labelSmall) },
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
        }

        // ── Grid de productos ──────────────────────────────────
        when {
            state.isLoading -> {
                Box(Modifier.fillMaxSize(), Alignment.Center) {
                    CircularProgressIndicator(color = Accent)
                }
            }
            state.error != null -> {
                Box(Modifier.fillMaxSize(), Alignment.Center) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Text("❌ ${state.error}", color = Error)
                        Spacer(Modifier.height(12.dp))
                        Button(
                            onClick = viewModel::refresh,
                            colors  = ButtonDefaults.buttonColors(containerColor = Accent),
                        ) { Text("Reintentar", color = AccentOnDark) }
                    }
                }
            }
            state.products.isEmpty() -> {
                Box(Modifier.fillMaxSize(), Alignment.Center) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Text("🔍", style = MaterialTheme.typography.displayMedium)
                        Spacer(Modifier.height(8.dp))
                        Text("Sin resultados", color = TextPrimary, fontWeight = FontWeight.Bold)
                        Text("Intenta con otra búsqueda", color = TextSecondary)
                    }
                }
            }
            else -> {
                LazyVerticalGrid(
                    columns       = GridCells.Fixed(2),
                    state         = gridState,
                    contentPadding = PaddingValues(16.dp),
                    verticalArrangement   = Arrangement.spacedBy(12.dp),
                    horizontalArrangement = Arrangement.spacedBy(12.dp),
                    modifier      = Modifier.fillMaxSize(),
                ) {
                    items(state.products, key = { it.id }) { product ->
                        ProductCard(
                            product = product,
                            onClick = { onProductClick(product.id) },
                        )
                    }

                    if (state.isLoadingMore) {
                        item(span = { GridItemSpan(2) }) {
                            Box(
                                modifier         = Modifier.fillMaxWidth().padding(16.dp),
                                contentAlignment = Alignment.Center,
                            ) {
                                CircularProgressIndicator(
                                    color       = Accent,
                                    modifier    = Modifier.size(28.dp),
                                    strokeWidth = 2.dp,
                                )
                            }
                        }
                    }
                }
            }
        }
    }
}
```

---

## 4.7 NavGraph completo — `presentation/navigation/NavGraph.kt`

```kotlin
// presentation/navigation/NavGraph.kt
package com.shopapp.presentation.navigation

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.navigation.NavType
import androidx.navigation.compose.*
import androidx.navigation.navArgument
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.ui.auth.LoginScreen
import com.shopapp.presentation.ui.auth.RegisterScreen
import com.shopapp.presentation.ui.uipublic.catalog.CatalogScreen
import com.shopapp.presentation.ui.uipublic.home.HomeScreen
import com.shopapp.presentation.viewmodel.AuthViewModel
import com.shopapp.presentation.viewmodel.CartViewModel
import com.shopapp.theme.Surface

@Composable
fun NavGraph(
    authViewModel: AuthViewModel,
    cartViewModel: CartViewModel = hiltViewModel(),
) {
    val isCheckingSession by authViewModel.isCheckingSession.collectAsState()

    if (isCheckingSession) {
        LoadingScreen("Iniciando ShopApp...")
        return
    }

    // Extraemos el contenido en un composable separado para que remember
    // y LaunchedEffect no queden condicionados por el early-return anterior.
    NavGraphContent(
        authViewModel = authViewModel,
        cartViewModel = cartViewModel,
    )
}

@Composable
private fun NavGraphContent(
    authViewModel: AuthViewModel,
    cartViewModel: CartViewModel,
) {
    val navController   = rememberNavController()
    val isAuthenticated by authViewModel.isAuthenticated.collectAsState()
    val isStaff         by authViewModel.isStaff.collectAsState()
    val cartCount       by cartViewModel.totalItems.collectAsState()

    // startDestination se fija UNA SOLA VEZ según el estado inicial de la sesión.
    // Si cambiara dinámicamente, NavHost recrearía el grafo y causaría el parpadeo.
    val startDestination = remember {
        when {
            !isAuthenticated -> Screen.Login.route
            isStaff          -> Screen.AdminDashboard.route
            else             -> Screen.Home.route
        }
    }

    // Cambios de auth POSTERIORES a la composición inicial se manejan aquí,
    // dentro de un efecto, nunca en el cuerpo de un composable.
    LaunchedEffect(isAuthenticated) {
        if (!isAuthenticated) {
            navController.navigate(Screen.Login.route) {
                popUpTo(0) { inclusive = true }
            }
        }
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
                    onCartClick   = { navController.navigate(Screen.Cart.route) },
                )
            }
        },
    ) { innerPadding ->

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
            ) {
                LoadingScreen("Detalle de producto — M5")
            }

            // ── CARRITO ────────────────────────────
            composable(Screen.Cart.route) {
                ScreenWithLogout(
                    title    = "Carrito — M5",
                    onLogout = { authViewModel.logout() },
                )
            }

            // ── PEDIDOS ────────────────────────────
            composable(Screen.Orders.route) {
                ScreenWithLogout(
                    title    = "Mis pedidos — M6",
                    onLogout = { authViewModel.logout() },
                )
            }

            // ── PERFIL ─────────────────────────────
            composable(Screen.Profile.route) {
                ScreenWithLogout(
                    title    = "Mi perfil — M6",
                    onLogout = { authViewModel.logout() },
                ) {
                    LoadingScreen("Mi perfil — M6")
                }
            }

            // ── ADMIN ──────────────────────────────
            composable(Screen.AdminDashboard.route) {
                ScreenWithLogout(
                    title    = "Admin Dashboard — M8",
                    onLogout = { authViewModel.logout() },
                )
            }
        }
    }
}

@Composable
fun ScreenWithLogout(
    title: String,
    onLogout: () -> Unit,
    content: @Composable () -> Unit = {},
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally,
    ) {

        content()

        Spacer(modifier = Modifier.height(16.dp))

        Text(
            text = title,
            style = MaterialTheme.typography.titleMedium,
        )

        Spacer(modifier = Modifier.height(24.dp))

        Button(onClick = onLogout) {
            Text("Cerrar sesión")
        }
    }
}


```

---

## 4.8 Actualizar `MainActivity.kt`

```kotlin
// MainActivity.kt
package com.shopapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material3.Surface
import androidx.compose.ui.Modifier
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.presentation.navigation.NavGraph
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
                    val authViewModel: AuthViewModel = hiltViewModel()
                    NavGraph(authViewModel = authViewModel)
                }
            }
        }
    }
}
```

---

## ✅ Checkpoint Módulo 4

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Arrancar la app | Splash brevemente, navega a Login o Home |
| 2 | Login con cliente | Home con Hero, categorías y productos |
| 3 | Tab "Catálogo" | Grid de productos en 2 columnas |
| 4 | Buscar "laptop" | Filtrado con debounce 400ms |
| 5 | Filtrar por categoría | Solo productos de esa categoría |
| 6 | Chip "Precio ↓" | Productos ordenados por precio descendente |
| 7 | Scroll al fondo del catálogo | Carga más productos automáticamente |
| 8 | Badge del carrito | Aparece al añadir ítems (M5) |
| 9 | Tab "Pedidos" sin login | Redirige a Login |
| 10 | Login con staff | Navega a Admin Dashboard (placeholder) |

---

## Resumen

| Elemento | Estado |
|---|---|
| `ProductRepositoryImpl` completo | ✅ |
| `RepositoryModule` actualizado | ✅ |
| `CartViewModel` con StateFlow | ✅ |
| `BottomNavBar` con badge de carrito | ✅ |
| `CatalogViewModel` con debounce y paginación | ✅ |
| `HomeScreen` con hero, categorías y novedades | ✅ |
| `CatalogScreen` con LazyVerticalGrid | ✅ |
| Filtros: búsqueda, categoría, ordenamiento | ✅ |
| `NavGraph` con Scaffold y guards | ✅ |
| `ProductCard` reutilizable | ✅ |

**Siguiente módulo →** M5: Detalle de producto, carrito con BottomSheet y checkoutimport androidx.compose.ui.unit.dp