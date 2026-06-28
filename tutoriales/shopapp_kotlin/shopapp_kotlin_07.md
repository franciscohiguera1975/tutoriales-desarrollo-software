# ShopApp Android — Módulo 7
## Admin — Scaffold con NavigationDrawer y Dashboard con KPIs

> **Objetivo:** Layout de administración con NavigationDrawer deslizable, TopAppBar contextual, Dashboard con KPIs en tiempo real cargados en paralelo y alertas de stock bajo.
> **Checkpoint final:** login con cuenta staff → Dashboard muestra métricas reales del backend, gráfica de barras por estado de pedido y lista de productos con stock bajo.

---

## 7.1 Repositorios de stats — agregar a los existentes

Los repositorios ya tienen `getStats()`. Necesitamos también el de usuarios.

### `data/repository/UserRepositoryImpl.kt`

```kotlin
// data/repository/UserRepositoryImpl.kt
package com.shopapp.data.repository

import com.shopapp.data.remote.api.UserApi
import com.shopapp.data.remote.dto.UserRequestDto
import com.shopapp.data.remote.dto.toDomain
import com.shopapp.data.remote.dto.toRequest
import com.shopapp.domain.model.User
import com.shopapp.domain.model.UserPayload
import com.shopapp.domain.repository.UserRepository
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class UserRepositoryImpl @Inject constructor(
    private val api: UserApi,
) : UserRepository {

    override suspend fun getUsers(
        search:   String?,
        isStaff:  Boolean?,
        isActive: Boolean?,
        page:     Int?,
    ): Result<Pair<List<User>, Int>> = runCatching {
        val response = api.getUsers(search, isStaff, isActive, page)
        if (response.isSuccessful) {
            val body = response.body()!!
            Pair(body.results.map { it.toDomain() }, body.count)
        } else error("Error ${response.code()}")
    }

    override suspend fun getUser(id: Int): Result<User> = runCatching {
        val response = api.getUser(id)
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}")
    }

    override suspend fun createUser(payload: UserPayload): Result<User> = runCatching {
        val response = api.createUser(payload.toRequest())
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}: ${response.errorBody()?.string()}")
    }

    override suspend fun updateUser(id: Int, payload: UserPayload): Result<User> = runCatching {
        val response = api.updateUser(id, payload.toRequest())
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}: ${response.errorBody()?.string()}")
    }

    override suspend fun deleteUser(id: Int): Result<Unit> = runCatching {
        val response = api.deleteUser(id)
        if (!response.isSuccessful) error("Error ${response.code()}")
    }

    override suspend fun toggleActive(id: Int): Result<Boolean> = runCatching {
        val response = api.toggleActive(id)
        if (response.isSuccessful) response.body()!!.isActive
        else error("Error ${response.code()}")
    }

    override suspend fun getStats(): Result<Map<String, Int>> = runCatching {
        val response = api.getStats()
        if (response.isSuccessful) {
            val s = response.body()!!
            mapOf(
                "total"    to s.total,
                "active"   to s.active,
                "inactive" to s.inactive,
                "staff"    to s.staff,
            )
        } else error("Error ${response.code()}")
    }
}
```

### `domain/repository/UserRepository.kt`

```kotlin
// domain/repository/UserRepository.kt
package com.shopapp.domain.repository

import com.shopapp.domain.model.User
import com.shopapp.domain.model.UserPayload

interface UserRepository {
    suspend fun getUsers(
        search:   String?  = null,
        isStaff:  Boolean? = null,
        isActive: Boolean? = null,
        page:     Int?     = null,
    ): Result<Pair<List<User>, Int>>
    suspend fun getUser(id: Int): Result<User>
    suspend fun createUser(payload: UserPayload): Result<User>
    suspend fun updateUser(id: Int, payload: UserPayload): Result<User>
    suspend fun deleteUser(id: Int): Result<Unit>
    suspend fun toggleActive(id: Int): Result<Boolean>
    suspend fun getStats(): Result<Map<String, Int>>
}
```

### Actualizar `di/RepositoryModule.kt` — bindings completos

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
    @Binds @Singleton abstract fun bindAuthRepository    (impl: AuthRepositoryImpl    ): AuthRepository
    @Binds @Singleton abstract fun bindCategoryRepository(impl: CategoryRepositoryImpl): CategoryRepository
    @Binds @Singleton abstract fun bindProductRepository (impl: ProductRepositoryImpl ): ProductRepository
    @Binds @Singleton abstract fun bindOrderRepository   (impl: OrderRepositoryImpl   ): OrderRepository
    @Binds @Singleton abstract fun bindUserRepository    (impl: UserRepositoryImpl    ): UserRepository
}
```

---

## 7.2 DashboardViewModel — `presentation/viewmodel/DashboardViewModel.kt`

```kotlin
// presentation/viewmodel/DashboardViewModel.kt
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.Product
import com.shopapp.domain.model.ProductFilters
import com.shopapp.domain.repository.*
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.async
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

data class DashboardStats(
    val totalActiveProducts:   Int    = 0,
    val outOfStockProducts:    Int    = 0,
    val totalStock:            Int    = 0,
    val avgPrice:              Double = 0.0,
    val activeCategories:      Int    = 0,
    val totalCategories:       Int    = 0,
    val totalOrders:           Int    = 0,
    val totalRevenue:          Double = 0.0,
    val pendingOrders:         Int    = 0,
    val ordersByStatus:        Map<String, Int> = emptyMap(),
    val activeUsers:           Int    = 0,
    val totalUsers:            Int    = 0,
    val staffUsers:            Int    = 0,
    val lowStockProducts:      List<Product> = emptyList(),
)

sealed interface DashboardUiState {
    data object Loading                          : DashboardUiState
    data class  Success(val stats: DashboardStats) : DashboardUiState
    data class  Error(val message: String)       : DashboardUiState
}

@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val productRepository:  ProductRepository,
    private val categoryRepository: CategoryRepository,
    private val orderRepository:    OrderRepository,
    private val userRepository:     UserRepository,
) : ViewModel() {

    private val _state = MutableStateFlow<DashboardUiState>(DashboardUiState.Loading)
    val state: StateFlow<DashboardUiState> = _state.asStateFlow()

    private val _lastUpdated = MutableStateFlow<Long>(0L)
    val lastUpdated: StateFlow<Long> = _lastUpdated.asStateFlow()

    init { load() }

    fun load() {
        viewModelScope.launch {
            _state.value = DashboardUiState.Loading

            try {
                // Todas las llamadas en paralelo con async
                val productStatsDeferred  = async { productRepository.getStats() }
                val categoryStatsDeferred = async { categoryRepository.getStats() }
                val orderStatsDeferred    = async { orderRepository.getStats() }
                val userStatsDeferred     = async { userRepository.getStats() }
                val lowStockDeferred      = async {
                    productRepository.getProducts(
                        ProductFilters(isActive = true, ordering = "stock", pageSize = 5)
                    )
                }

                val productStats  = productStatsDeferred.await().getOrThrow()
                val categoryStats = categoryStatsDeferred.await().getOrThrow()
                val orderStats    = orderStatsDeferred.await().getOrThrow()
                val userStats     = userStatsDeferred.await().getOrThrow()
                val lowStock      = lowStockDeferred.await().getOrNull()

                @Suppress("UNCHECKED_CAST")
                val ordersByStatus = (orderStats["by_status"] as? Map<String, Int>) ?: emptyMap()

                val stats = DashboardStats(
                    totalActiveProducts  = (productStats["total_active"]   as? Int)    ?: 0,
                    outOfStockProducts   = (productStats["out_of_stock"]   as? Int)    ?: 0,
                    totalStock           = (productStats["total_stock"]    as? Int)    ?: 0,
                    avgPrice             = (productStats["avg_price"]      as? Double) ?: 0.0,
                    activeCategories     = (categoryStats["active"]        as? Int)    ?: 0,
                    totalCategories      = (categoryStats["total"]         as? Int)    ?: 0,
                    totalOrders          = (orderStats["total_orders"]     as? Int)    ?: 0,
                    totalRevenue         = (orderStats["total_revenue"]    as? Double) ?: 0.0,
                    pendingOrders        = ordersByStatus["pending"]                   ?: 0,
                    ordersByStatus       = ordersByStatus,
                    activeUsers          = (userStats["active"]            as? Int)    ?: 0,
                    totalUsers           = (userStats["total"]             as? Int)    ?: 0,
                    staffUsers           = (userStats["staff"]             as? Int)    ?: 0,
                    lowStockProducts     = lowStock?.first
                        ?.filter { it.stock < 5 }
                        ?.take(5)
                        ?: emptyList(),
                )

                _state.value       = DashboardUiState.Success(stats)
                _lastUpdated.value = System.currentTimeMillis()

            } catch (e: Exception) {
                _state.value = DashboardUiState.Error(e.message ?: "Error al cargar el dashboard")
            }
        }
    }
}
```
data/repository/CategoryRepositoryImpl.kt

```kotlin
// data/repository/CategoryRepositoryImpl.kt
package com.shopapp.data.repository

import com.shopapp.data.remote.api.CategoryApi
import com.shopapp.data.remote.dto.toDomain
import com.shopapp.data.remote.dto.toRequest
import com.shopapp.domain.model.Category
import com.shopapp.domain.model.CategoryPayload
import com.shopapp.domain.repository.CategoryRepository
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class CategoryRepositoryImpl @Inject constructor(
    private val api: CategoryApi,
) : CategoryRepository {

    override suspend fun getCategories(): Result<List<Category>> = runCatching {
        val response = api.getCategories()
        if (response.isSuccessful) {
            response.body()!!.results.map { it.toDomain() }
        } else {
            error("Error ${response.code()}: ${response.errorBody()?.string()}")
        }
    }

    override suspend fun getCategory(id: Int): Result<Category> = runCatching {
        val response = api.getCategory(id)
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}")
    }

    override suspend fun createCategory(payload: CategoryPayload): Result<Category> = runCatching {
        val response = api.createCategory(payload.toRequest())
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}: ${response.errorBody()?.string()}")
    }

    override suspend fun updateCategory(id: Int, payload: CategoryPayload): Result<Category> =
        runCatching {
            val response = api.updateCategory(id, payload.toRequest())
            if (response.isSuccessful) response.body()!!.toDomain()
            else error("Error ${response.code()}: ${response.errorBody()?.string()}")
        }

    override suspend fun deleteCategory(id: Int): Result<Unit> = runCatching {
        val response = api.deleteCategory(id)
        if (!response.isSuccessful) {
            error("Error ${response.code()}: ${response.errorBody()?.string()}")
        }
    }

    override suspend fun getStats(): Result<Map<String, Any>> = runCatching {
        val response = api.getStats()
        if (response.isSuccessful) {
            val s = response.body()!!

            mapOf(
                "total"    to s.total,
                "active"   to s.active,
                "inactive" to s.inactive,
                "detail"   to s.detail // lista de categorías con num_products
            )
        } else {
            error("Error ${response.code()}: ${response.errorBody()?.string()}")
        }
    }
}

```

domain/repository/CategoryRepository.kt

```kotlin
// domain/repository/CategoryRepository.kt
package com.shopapp.domain.repository

import com.shopapp.domain.model.Category
import com.shopapp.domain.model.CategoryPayload

interface CategoryRepository {
    suspend fun getCategories(): Result<List<Category>>
    suspend fun getCategory(id: Int): Result<Category>
    suspend fun createCategory(payload: CategoryPayload): Result<Category>
    suspend fun updateCategory(id: Int, payload: CategoryPayload): Result<Category>
    suspend fun deleteCategory(id: Int): Result<Unit>
    suspend fun getStats(): Result<Map<String, Any>>
}

```

data/remote/dto/CategoryDto.kt
```kotlin
// data/remote/dto/CategoryDto.kt
package com.shopapp.data.remote.dto

import com.google.gson.annotations.SerializedName
import com.shopapp.domain.model.Category
import com.shopapp.domain.model.CategoryPayload

data class CategoryDto(
    val id:          Int,
    val name:        String,
    val slug:        String,
    val description: String,
    @SerializedName("is_active")      val isActive:      Boolean,
    @SerializedName("total_products") val totalProducts: Int,
    @SerializedName("created_at")     val createdAt:     String,
)

data class CategoryRequestDto(
    val name:        String,
    val slug:        String,
    val description: String,
    @SerializedName("is_active") val isActive: Boolean,
)

data class CategoryStatsDto(
    val total: Int,
    val active: Int,
    val inactive: Int,
    val detail: List<CategoryDetailDto>
)

data class CategoryDetailDto(
    val id: Int,
    val name: String,
    val num_products: Int,
    val is_active: Boolean
)

// ── Mappers ───────────────────────────────────────────────────

fun CategoryDto.toDomain() = Category(
    id            = id,
    name          = name,
    slug          = slug,
    description   = description,
    isActive      = isActive,
    totalProducts = totalProducts,
    createdAt     = createdAt,
)

fun CategoryPayload.toRequest() = CategoryRequestDto(
    name        = name,
    slug        = slug,
    description = description,
    isActive    = isActive,
)

```

---

## 7.4 KpiCard — `presentation/ui/admin/dashboard/KpiCard.kt`

```kotlin
// presentation/ui/admin/dashboard/KpiCard.kt
package com.shopapp.presentation.ui.admin.dashboard

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.shopapp.theme.*

@Composable
fun KpiCard(
    title:     String,
    value:     String,
    subtitle:  String?     = null,
    icon:      ImageVector,
    color:     Color       = Accent,
    hasAlert:  Boolean     = false,
    onClick:   (() -> Unit)? = null,
    modifier:  Modifier    = Modifier,
) {
    val container: @Composable (@Composable () -> Unit) -> Unit = { content ->
        if (onClick != null) {
            Surface(
                onClick        = onClick,
                shape          = MaterialTheme.shapes.large,
                color          = Surface,
                tonalElevation = 0.dp,
                modifier       = modifier.fillMaxWidth(),
            ) { content() }
        } else {
            Surface(
                shape          = MaterialTheme.shapes.large,
                color          = Surface,
                tonalElevation = 0.dp,
                modifier       = modifier.fillMaxWidth(),
            ) { content() }
        }
    }

    container {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .then(
                    Modifier.background(
                        color  = color.copy(alpha = 0.06f),
                        shape  = MaterialTheme.shapes.large,
                    )
                )
                .padding(16.dp),
        ) {
            Row(
                modifier              = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.Top,
            ) {
                Box(
                    modifier         = Modifier
                        .size(40.dp)
                        .background(color.copy(alpha = 0.15f), MaterialTheme.shapes.medium),
                    contentAlignment = Alignment.Center,
                ) {
                    Icon(icon, contentDescription = null, tint = color, modifier = Modifier.size(22.dp))
                }
                if (hasAlert) {
                    Text("⚠️", fontSize = 18.sp)
                }
            }

            Spacer(Modifier.height(12.dp))

            Text(
                text       = value,
                fontSize   = 28.sp,
                fontWeight = FontWeight.ExtraBold,
                color      = TextPrimary,
                lineHeight = 30.sp,
            )
            Text(
                text  = title,
                style = MaterialTheme.typography.bodySmall,
                color = TextSecondary,
                fontWeight = FontWeight.Medium,
            )
            if (subtitle != null) {
                Text(
                    text  = subtitle,
                    style = MaterialTheme.typography.bodySmall.copy(fontSize = 11.sp),
                    color = if (hasAlert) Error else TextFaint,
                )
            }
        }
    }
}
```

---

## 7.3 AdminScaffold con NavigationDrawer — `presentation/ui/admin/AdminScaffold.kt`

```kotlin
// presentation/ui/admin/AdminScaffold.kt
package com.shopapp.presentation.ui.admin

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.vector.ImageVector
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.shopapp.domain.model.LoggedUser
import com.shopapp.theme.*
import kotlinx.coroutines.launch

data class AdminNavItem(
    val label:  String,
    val icon:   ImageVector,
    val route:  String,
)

val ADMIN_NAV_ITEMS = listOf(
    AdminNavItem("Dashboard",  Icons.Default.Dashboard,     "admin"),
    AdminNavItem("Categorías", Icons.Default.Category,      "admin/categories"),
    AdminNavItem("Productos",  Icons.Default.Inventory,     "admin/products"),
    AdminNavItem("Pedidos",    Icons.Default.ShoppingBag,   "admin/orders"),
    AdminNavItem("Usuarios",   Icons.Default.People,        "admin/users"),
)

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun AdminScaffold(
    currentRoute:  String,
    user:          LoggedUser?,
    onNavClick:    (String) -> Unit,
    onStoreClick:  () -> Unit,
    onLogout:      () -> Unit,
    title:         String,
    content:       @Composable (PaddingValues) -> Unit,
) {
    val drawerState = rememberDrawerState(DrawerValue.Closed)
    val scope       = rememberCoroutineScope()

    ModalNavigationDrawer(
        drawerState    = drawerState,
        drawerContent  = {
            AdminDrawerContent(
                currentRoute  = currentRoute,
                user          = user,
                onNavClick    = { route ->
                    scope.launch { drawerState.close() }
                    onNavClick(route)
                },
                onStoreClick  = {
                    scope.launch { drawerState.close() }
                    onStoreClick()
                },
                onLogout      = {
                    scope.launch { drawerState.close() }
                    onLogout()
                },
            )
        },
    ) {
        Scaffold(
            topBar = {
                TopAppBar(
                    title   = {
                        Text(
                            text       = title,
                            style      = MaterialTheme.typography.titleMedium,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                        )
                    },
                    navigationIcon = {
                        IconButton(onClick = { scope.launch { drawerState.open() } }) {
                            Icon(Icons.Default.Menu, contentDescription = "Menú", tint = TextPrimary)
                        }
                    },
                    actions = {
                        TextButton(onClick = onStoreClick) {
                            Text(
                                "← Tienda",
                                color = Accent,
                                style = MaterialTheme.typography.bodySmall,
                                fontWeight = FontWeight.SemiBold,
                            )
                        }
                    },
                    colors = TopAppBarDefaults.topAppBarColors(containerColor = Surface),
                )
            },
            containerColor = Background,
            content        = content,
        )
    }
}

@Composable
private fun AdminDrawerContent(
    currentRoute: String,
    user:         LoggedUser?,
    onNavClick:   (String) -> Unit,
    onStoreClick: () -> Unit,
    onLogout:     () -> Unit,
) {
    ModalDrawerSheet(
        drawerContainerColor = Surface,
        modifier             = Modifier.width(280.dp),
    ) {
        // Header con logo
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .background(Surface2)
                .padding(24.dp),
        ) {
            Text(
                text       = "ShopApp",
                fontSize   = 22.sp,
                fontWeight = FontWeight.Bold,
                color      = Accent,
            )
            Text(
                text  = "Panel de administración",
                style = MaterialTheme.typography.bodySmall,
                color = TextSecondary,
            )
        }

        HorizontalDivider(color = Border, thickness = 0.5.dp)

        // Avatar del admin
        if (user != null) {
            Row(
                modifier          = Modifier
                    .fillMaxWidth()
                    .padding(16.dp),
                verticalAlignment = Alignment.CenterVertically,
            ) {
                Box(
                    modifier         = Modifier
                        .size(40.dp)
                        .background(
                            brush = androidx.compose.ui.graphics.Brush.linearGradient(
                                listOf(Accent, AccentLight)
                            ),
                            shape = CircleShape,
                        ),
                    contentAlignment = Alignment.Center,
                ) {
                    Text(
                        text       = user.username.firstOrNull()?.uppercaseChar()?.toString() ?: "A",
                        color      = AccentOnDark,
                        fontWeight = FontWeight.Bold,
                        fontSize   = 16.sp,
                    )
                }
                Spacer(Modifier.width(12.dp))
                Column {
                    Text(
                        text       = user.username,
                        style      = MaterialTheme.typography.bodyMedium,
                        fontWeight = FontWeight.SemiBold,
                        color      = TextPrimary,
                    )
                    Surface(
                        color  = Accent.copy(alpha = 0.15f),
                        shape  = MaterialTheme.shapes.extraSmall,
                    ) {
                        Text(
                            text       = "Staff",
                            color      = Accent,
                            fontSize   = 10.sp,
                            fontWeight = FontWeight.Bold,
                            modifier   = Modifier.padding(horizontal = 8.dp, vertical = 2.dp),
                        )
                    }
                }
            }
        }

        HorizontalDivider(color = Border, thickness = 0.5.dp)
        Spacer(Modifier.height(8.dp))

        // Items de navegación
        ADMIN_NAV_ITEMS.forEach { item ->
            val isSelected = currentRoute == item.route || currentRoute.startsWith("${item.route}/")
            NavigationDrawerItem(
                icon     = {
                    Icon(
                        item.icon,
                        contentDescription = item.label,
                        tint = if (isSelected) Accent else TextSecondary,
                    )
                },
                label    = {
                    Text(
                        text       = item.label,
                        color      = if (isSelected) Accent else TextSecondary,
                        fontWeight = if (isSelected) FontWeight.Bold else FontWeight.Normal,
                    )
                },
                selected = isSelected,
                onClick  = { onNavClick(item.route) },
                colors   = NavigationDrawerItemDefaults.colors(
                    selectedContainerColor   = Accent.copy(alpha = 0.12f),
                    unselectedContainerColor = androidx.compose.ui.graphics.Color.Transparent,
                ),
                modifier = Modifier.padding(horizontal = 12.dp, vertical = 2.dp),
            )
        }

        Spacer(Modifier.weight(1f))
        HorizontalDivider(color = Border, thickness = 0.5.dp)

        // Cerrar sesión
        NavigationDrawerItem(
            icon     = { Icon(Icons.Default.Logout, contentDescription = "Salir", tint = Error) },
            label    = {
                Text("Cerrar sesión", color = Error, fontWeight = FontWeight.SemiBold)
            },
            selected = false,
            onClick  = onLogout,
            colors   = NavigationDrawerItemDefaults.colors(
                unselectedContainerColor = androidx.compose.ui.graphics.Color.Transparent,
            ),
            modifier = Modifier.padding(horizontal = 12.dp, vertical = 8.dp),
        )
    }
}
```


---

## 7.5 DashboardScreen — `presentation/ui/admin/dashboard/DashboardScreen.kt`

```kotlin
// presentation/ui/admin/dashboard/DashboardScreen.kt
package com.shopapp.presentation.ui.admin.dashboard

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
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.domain.model.OrderStatus
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.components.orderStatusColor
import com.shopapp.presentation.viewmodel.DashboardUiState
import com.shopapp.presentation.viewmodel.DashboardViewModel
import com.shopapp.theme.*
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

@Composable
fun DashboardScreen(
    onNavigate: (String) -> Unit,
    viewModel:  DashboardViewModel = hiltViewModel(),
) {
    val state       by viewModel.state.collectAsState()
    val lastUpdated by viewModel.lastUpdated.collectAsState()

    when (val s = state) {
        is DashboardUiState.Loading ->
            LoadingScreen("Cargando dashboard...")
        is DashboardUiState.Error   -> {
            Box(Modifier.fillMaxSize(), Alignment.Center) {
                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                    Text("⚠️ ${s.message}", color = Error)
                    Spacer(Modifier.height(12.dp))
                    Button(onClick = viewModel::load,
                        colors = ButtonDefaults.buttonColors(containerColor = Accent)) {
                        Text("Reintentar", color = AccentOnDark)
                    }
                }
            }
        }
        is DashboardUiState.Success ->
            DashboardContent(
                stats       = s.stats,
                lastUpdated = lastUpdated,
                onNavigate  = onNavigate,
                onRefresh   = viewModel::load,
            )
    }
}

@Composable
private fun DashboardContent(
    stats:       com.shopapp.presentation.viewmodel.DashboardStats,
    lastUpdated: Long,
    onNavigate:  (String) -> Unit,
    onRefresh:   () -> Unit,
) {
    val timeFmt = SimpleDateFormat("HH:mm:ss", Locale.getDefault())
    val timeStr = if (lastUpdated > 0) timeFmt.format(Date(lastUpdated)) else "—"

    LazyColumn(
        modifier       = Modifier.fillMaxSize().background(Background),
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp),
    ) {
        // Header
        item {
            Row(
                modifier              = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.CenterVertically,
            ) {
                Column {
                    Text(
                        text       = "Dashboard",
                        style      = MaterialTheme.typography.headlineMedium,
                        fontWeight = FontWeight.Bold,
                        color      = TextPrimary,
                    )
                    Text(
                        text  = "Actualizado: $timeStr",
                        style = MaterialTheme.typography.bodySmall,
                        color = TextFaint,
                    )
                }
                IconButton(onClick = onRefresh) {
                    Icon(Icons.Default.Refresh, contentDescription = "Actualizar", tint = Accent)
                }
            }
        }

        // ── KPIs — fila 1 ─────────────────────────────────────
        item {
            Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                KpiCard(
                    title    = "Productos activos",
                    value    = stats.totalActiveProducts.toString(),
                    subtitle = if (stats.outOfStockProducts > 0)
                               "${stats.outOfStockProducts} sin stock" else null,
                    icon     = Icons.Default.Inventory,
                    color    = Accent,
                    hasAlert = stats.outOfStockProducts > 0,
                    onClick  = { onNavigate("admin/products") },
                    modifier = Modifier.weight(1f),
                )
                KpiCard(
                    title   = "Categorías activas",
                    value   = stats.activeCategories.toString(),
                    subtitle = "${stats.totalCategories} total",
                    icon    = Icons.Default.Category,
                    color   = Info,
                    onClick = { onNavigate("admin/categories") },
                    modifier = Modifier.weight(1f),
                )
            }
        }

        // ── KPIs — fila 2 ─────────────────────────────────────
        item {
            Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                KpiCard(
                    title    = "Pedidos totales",
                    value    = stats.totalOrders.toString(),
                    subtitle = if (stats.pendingOrders > 0)
                               "${stats.pendingOrders} pendientes" else null,
                    icon     = Icons.Default.ShoppingBag,
                    color    = Success,
                    hasAlert = stats.pendingOrders > 0,
                    onClick  = { onNavigate("admin/orders") },
                    modifier = Modifier.weight(1f),
                )
                KpiCard(
                    title    = "Usuarios activos",
                    value    = stats.activeUsers.toString(),
                    subtitle = "${stats.totalUsers} registrados",
                    icon     = Icons.Default.People,
                    color    = Warning,
                    onClick  = { onNavigate("admin/users") },
                    modifier = Modifier.weight(1f),
                )
            }
        }

        // ── KPIs financieros ──────────────────────────────────
        item {
            Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
                KpiCard(
                    title    = "Facturación total",
                    value    = "$${"%.0f".format(stats.totalRevenue)}",
                    icon     = Icons.Default.TrendingUp,
                    color    = Accent,
                    modifier = Modifier.weight(1f),
                )
                KpiCard(
                    title    = "Precio medio",
                    value    = "$${"%.2f".format(stats.avgPrice)}",
                    subtitle = "por producto",
                    icon     = Icons.Default.Sell,
                    color    = TextSecondary,
                    modifier = Modifier.weight(1f),
                )
            }
        }

        // ── Pedidos por estado — gráfica de barras ─────────────
        if (stats.ordersByStatus.isNotEmpty()) {
            item {
                Surface(
                    color    = Surface,
                    shape    = MaterialTheme.shapes.large,
                    modifier = Modifier.fillMaxWidth(),
                ) {
                    Column(modifier = Modifier.padding(16.dp)) {
                        Row(
                            modifier              = Modifier.fillMaxWidth(),
                            horizontalArrangement = Arrangement.SpaceBetween,
                            verticalAlignment     = Alignment.CenterVertically,
                        ) {
                            Text(
                                text       = "Pedidos por estado",
                                style      = MaterialTheme.typography.titleSmall,
                                fontWeight = FontWeight.Bold,
                                color      = TextPrimary,
                            )
                            TextButton(onClick = { onNavigate("admin/orders") }) {
                                Text("Ver todos", color = Accent,
                                    style = MaterialTheme.typography.bodySmall)
                            }
                        }
                        Spacer(Modifier.height(16.dp))

                        val total = stats.totalOrders.coerceAtLeast(1)
                        stats.ordersByStatus.entries.forEach { (statusValue, count) ->
                            val status = OrderStatus.fromValue(statusValue)
                            val color  = orderStatusColor(status)
                            val pct    = (count.toFloat() / total).coerceIn(0.02f, 1f)

                            Column(modifier = Modifier.padding(bottom = 10.dp)) {
                                Row(
                                    modifier              = Modifier.fillMaxWidth(),
                                    horizontalArrangement = Arrangement.SpaceBetween,
                                ) {
                                    Text(
                                        text  = status.label,
                                        style = MaterialTheme.typography.bodySmall,
                                        color = TextSecondary,
                                    )
                                    Text(
                                        text       = count.toString(),
                                        style      = MaterialTheme.typography.bodySmall,
                                        fontWeight = FontWeight.Bold,
                                        color      = color,
                                    )
                                }
                                Spacer(Modifier.height(4.dp))
                                Box(
                                    modifier = Modifier
                                        .fillMaxWidth()
                                        .height(7.dp)
                                        .background(Surface2, MaterialTheme.shapes.extraSmall),
                                ) {
                                    Box(
                                        modifier = Modifier
                                            .fillMaxWidth(pct)
                                            .fillMaxHeight()
                                            .background(color, MaterialTheme.shapes.extraSmall),
                                    )
                                }
                            }
                        }
                    }
                }
            }
        }

        // ── Productos con stock bajo ───────────────────────────
        item {
            Surface(
                color    = Surface,
                shape    = MaterialTheme.shapes.large,
                modifier = Modifier.fillMaxWidth(),
            ) {
                Column(modifier = Modifier.padding(16.dp)) {
                    Row(
                        modifier              = Modifier.fillMaxWidth(),
                        horizontalArrangement = Arrangement.SpaceBetween,
                        verticalAlignment     = Alignment.CenterVertically,
                    ) {
                        Row(verticalAlignment = Alignment.CenterVertically) {
                            Icon(
                                Icons.Default.Warning,
                                contentDescription = null,
                                tint    = Warning,
                                modifier = Modifier.size(18.dp),
                            )
                            Spacer(Modifier.width(6.dp))
                            Text(
                                text       = "Stock bajo",
                                style      = MaterialTheme.typography.titleSmall,
                                fontWeight = FontWeight.Bold,
                                color      = TextPrimary,
                            )
                        }
                        TextButton(onClick = { onNavigate("admin/products") }) {
                            Text("Gestionar", color = Accent,
                                style = MaterialTheme.typography.bodySmall)
                        }
                    }

                    if (stats.lowStockProducts.isEmpty()) {
                        Box(
                            modifier         = Modifier.fillMaxWidth().padding(16.dp),
                            contentAlignment = Alignment.Center,
                        ) {
                            Text("✅ Sin alertas de stock", color = Success,
                                style = MaterialTheme.typography.bodySmall)
                        }
                    } else {
                        Spacer(Modifier.height(8.dp))
                        stats.lowStockProducts.forEach { product ->
                            Surface(
                                onClick  = { onNavigate("admin/products") },
                                color    = Surface2,
                                shape    = MaterialTheme.shapes.medium,
                                modifier = Modifier.fillMaxWidth().padding(bottom = 6.dp),
                            ) {
                                Row(
                                    modifier              = Modifier
                                        .fillMaxWidth()
                                        .padding(horizontal = 14.dp, vertical = 10.dp),
                                    horizontalArrangement = Arrangement.SpaceBetween,
                                    verticalAlignment     = Alignment.CenterVertically,
                                ) {
                                    Text(
                                        text     = product.name,
                                        style    = MaterialTheme.typography.bodySmall,
                                        color    = TextPrimary,
                                        fontWeight = FontWeight.Medium,
                                        modifier = Modifier.weight(1f),
                                        maxLines = 1,
                                    )
                                    Surface(
                                        color = if (product.stock == 0)
                                                    Error.copy(alpha = 0.15f)
                                                else Warning.copy(alpha = 0.15f),
                                        shape = MaterialTheme.shapes.extraSmall,
                                    ) {
                                        Text(
                                            text       = if (product.stock == 0) "Agotado"
                                                         else "${product.stock} uds.",
                                            color      = if (product.stock == 0) Error else Warning,
                                            fontSize   = 11.sp,
                                            fontWeight = FontWeight.Bold,
                                            modifier   = Modifier.padding(horizontal = 8.dp, vertical = 3.dp),
                                        )
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }

        // ── Acciones rápidas ──────────────────────────────────
        item {
            Surface(
                color    = Surface,
                shape    = MaterialTheme.shapes.large,
                modifier = Modifier.fillMaxWidth(),
            ) {
                Column(modifier = Modifier.padding(16.dp)) {
                    Text(
                        text       = "⚡ Acciones rápidas",
                        style      = MaterialTheme.typography.titleSmall,
                        fontWeight = FontWeight.Bold,
                        color      = TextPrimary,
                        modifier   = Modifier.padding(bottom = 12.dp),
                    )
                    LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                        items(listOf(
                            Triple("+ Categoría", Info,    "admin/categories"),
                            Triple("+ Producto",  Accent,  "admin/products"),
                            Triple("Ver pedidos", Success, "admin/orders"),
                            Triple("Usuarios",    Warning, "admin/users"),
                        )) { (label, color, route) ->
                            Surface(
                                onClick  = { onNavigate(route) },
                                color    = color.copy(alpha = 0.1f),
                                shape    = MaterialTheme.shapes.medium,
                            ) {
                                Text(
                                    text       = label,
                                    color      = color,
                                    fontWeight = FontWeight.Bold,
                                    style      = MaterialTheme.typography.bodySmall,
                                    modifier   = Modifier.padding(horizontal = 14.dp, vertical = 10.dp),
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

## 7.6 NavGraph admin — `presentation/navigation/NavGraph.kt`

Actualizar el NavHost para incluir las rutas admin con el `AdminScaffold`:

```kotlin
// NavGraph.kt — dentro de @Composable NavGraph()
// presentation/navigation/NavGraph.kt
package com.shopapp.presentation.navigation

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.navigation.NavType
import androidx.navigation.compose.*
import androidx.navigation.navArgument
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.ui.admin.AdminScaffold
import com.shopapp.presentation.ui.admin.DashboardScreen
import com.shopapp.presentation.ui.auth.LoginScreen
import com.shopapp.presentation.ui.auth.RegisterScreen
import com.shopapp.presentation.ui.uipublic.catalog.CatalogScreen
import com.shopapp.presentation.ui.uipublic.home.HomeScreen
import com.shopapp.presentation.ui.uipublic.product.ProductDetailScreen
import com.shopapp.presentation.ui.uipublic.cart.CartBottomSheet
import com.shopapp.presentation.ui.client.orders.OrdersScreen
import com.shopapp.presentation.ui.client.orders.OrderDetailScreen
import com.shopapp.presentation.ui.client.profile.ProfileScreen
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
                    onCartClick   = { showCart = true }, // 🔥 clave
                )
            }
        },
    ) { innerPadding ->

        // 🔥 BottomSheet del carrito
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

            // ── PEDIDOS ────────────────────────────
            composable(Screen.Orders.route) {
                if (!isAuthenticated) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Login.route) { popUpTo(Screen.Home.route) }
                    }
                } else {
                    OrdersScreen(
                        onOrderClick = { id -> navController.navigate("orders/$id") },
                    )
                }
            }

            // ── DETALLE PEDIDO ─────────────────────
            composable(
                route     = Screen.OrderDetail().route,
                arguments = listOf(navArgument("id") { type = NavType.IntType }),
            ) { backStackEntry ->
                val id = backStackEntry.arguments?.getInt("id") ?: return@composable
                OrderDetailScreen(
                    orderId = id,
                    onBack  = { navController.popBackStack() },
                )
            }

            // ── PERFIL ─────────────────────────────
            composable(Screen.Profile.route) {
                if (!isAuthenticated) {
                    LaunchedEffect(Unit) {
                        navController.navigate(Screen.Login.route) { popUpTo(Screen.Home.route) }
                    }
                } else {
                    ProfileScreen(
                        authViewModel = authViewModel,
                        onLogout      = {
                            navController.navigate(Screen.Login.route) {
                                popUpTo(0) { inclusive = true }
                            }
                        },
                    )
                }
            }

            // ── ADMIN ──────────────────────────────
            composable(Screen.AdminDashboard.route) {
                if (!isStaff) {
                    LaunchedEffect(Unit) { navController.navigate(Screen.Home.route) { popUpTo(0) } }
                    return@composable
                }
                AdminScaffold(
                    currentRoute = Screen.AdminDashboard.route,
                    user         = authViewModel.currentUser.collectAsState().value,
                    title        = "Dashboard",
                    onNavClick   = { route -> navController.navigate(route) {
                        launchSingleTop = true
                        restoreState    = true
                    }},
                    onStoreClick = {
                        navController.navigate(Screen.Home.route) {
                            popUpTo(Screen.AdminDashboard.route) { inclusive = false }
                        }
                    },
                    onLogout     = {
                        authViewModel.logout()
                        navController.navigate(Screen.Login.route) { popUpTo(0) { inclusive = true } }
                    },
                ) { padding ->
                    Box(modifier = Modifier.padding(padding)) {
                        DashboardScreen(onNavigate = { route -> navController.navigate(route) })
                    }
                }
            }
            // Placeholders para M8-M12 — misma estructura con AdminScaffold
            listOf(
                "admin/categories" to "Categorías",
                "admin/products"   to "Productos",
                "admin/orders"     to "Pedidos",
                "admin/users"      to "Usuarios",
            ).forEach { (route, title) ->
                composable(route) {
                    if (!isStaff) {
                        LaunchedEffect(Unit) { navController.navigate(Screen.Home.route) { popUpTo(0) } }
                        return@composable
                    }
                    AdminScaffold(
                        currentRoute = route,
                        user         = authViewModel.currentUser.collectAsState().value,
                        title        = title,
                        onNavClick   = { r -> navController.navigate(r) { launchSingleTop = true } },
                        onStoreClick = { navController.navigate(Screen.Home.route) },
                        onLogout     = {
                            authViewModel.logout()
                            navController.navigate(Screen.Login.route) { popUpTo(0) { inclusive = true } }
                        },
                    ) { padding ->
                        Box(
                            modifier         = Modifier.fillMaxSize().padding(padding),
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

## ✅ Checkpoint Módulo 7

### Preparar datos de prueba en Django

```bash
uv run python manage.py shell -c "
from store.models import Category, Product
cat = Category.objects.first()
if cat:
    Product.objects.get_or_create(
        name='Producto agotado', defaults={'price': 30, 'stock': 0, 'category': cat}
    )
    Product.objects.get_or_create(
        name='Stock bajo', defaults={'price': 50, 'stock': 2, 'category': cat}
    )
print('Datos de prueba listos')
"
```

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Login con cuenta staff | Navega al Dashboard |
| 2 | Dashboard carga | 6 KPIs con datos reales |
| 3 | KPI "Sin stock" | Alerta ⚠️ y texto rojo |
| 4 | Tap en KPI | Navega a la sección correspondiente |
| 5 | Gráfica de barras | Barras proporcionales por estado |
| 6 | "Stock bajo" | Productos con stock < 5 |
| 7 | Abrir drawer (☰) | Panel deslizable con 5 secciones |
| 8 | Navegar a "Categorías" | Placeholder "próximo módulo" |
| 9 | Botón "← Tienda" | Navega a Home |
| 10 | Logout desde el drawer | Regresa a Login |

---

## Resumen

| Elemento | Estado |
|---|---|
| `UserRepositoryImpl` completo | ✅ |
| `RepositoryModule` con los 5 repositorios | ✅ |
| `DashboardViewModel` con `async` paralelo | ✅ |
| `AdminScaffold` con `ModalNavigationDrawer` | ✅ |
| `AdminDrawerContent` con avatar y nav items | ✅ |
| `KpiCard` clickeable con alerta | ✅ |
| `DashboardScreen` con 6 KPIs en pares | ✅ |
| Gráfica de barras CSS pura por estado | ✅ |
| Lista de productos con stock bajo | ✅ |
| Acciones rápidas horizontales | ✅ |
| Rutas admin con guard `isStaff` | ✅ |

**Siguiente módulo →** M8: Admin CRUD Categorías — lista, crear, editar y toggle optimista