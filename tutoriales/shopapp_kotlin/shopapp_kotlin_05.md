# ShopApp Android — Módulo 5
## Detalle de producto, Carrito con BottomSheet y Checkout

> **Objetivo:** Pantalla de detalle de producto con selector de cantidad, carrito como BottomSheet deslizable y flujo de checkout que crea el pedido real en Django.
> **Checkpoint final:** añadir un producto al carrito, ver el BottomSheet con totales e IVA, confirmar el pedido y verificarlo en el admin de Django.

---

## 5.1 Repositorio de pedidos — `domain/repository/OrderRepository.kt`

```kotlin
// domain/repository/OrderRepository.kt
package com.shopapp.domain.repository

import com.shopapp.domain.model.Order
import com.shopapp.domain.model.OrderStatus

interface OrderRepository {
    suspend fun getOrders(page: Int? = null, status: String? = null): Result<Pair<List<Order>, Int>>
    suspend fun getOrder(id: Int): Result<Order>
    suspend fun createOrder(): Result<Order>
    suspend fun addItem(orderId: Int, productId: Int, quantity: Int): Result<Order>
    suspend fun confirmOrder(orderId: Int): Result<Order>
    suspend fun updateStatus(orderId: Int, status: OrderStatus): Result<Order>
    suspend fun getStats(): Result<Map<String, Any>>
}
```

### `data/repository/OrderRepositoryImpl.kt`

```kotlin
// data/repository/OrderRepositoryImpl.kt
package com.shopapp.data.repository

import com.shopapp.data.remote.api.OrderApi
import com.shopapp.data.remote.dto.AddItemRequestDto
import com.shopapp.data.remote.dto.UpdateStatusRequestDto
import com.shopapp.data.remote.dto.toDomain
import com.shopapp.domain.model.Order
import com.shopapp.domain.model.OrderStatus
import com.shopapp.domain.repository.OrderRepository
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class OrderRepositoryImpl @Inject constructor(
    private val api: OrderApi,
) : OrderRepository {

    override suspend fun getOrders(page: Int?, status: String?): Result<Pair<List<Order>, Int>> =
        runCatching {
            val response = api.getOrders(page = page, status = status)
            if (response.isSuccessful) {
                val body = response.body()!!
                Pair(body.results.map { it.toDomain() }, body.count)
            } else error("Error ${response.code()}")
        }

    override suspend fun getOrder(id: Int): Result<Order> = runCatching {
        val response = api.getOrder(id)
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}")
    }

    override suspend fun createOrder(): Result<Order> = runCatching {
        val response = api.createOrder()
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}: ${response.errorBody()?.string()}")
    }

    override suspend fun addItem(orderId: Int, productId: Int, quantity: Int): Result<Order> =
        runCatching {
            val response = api.addItem(orderId, AddItemRequestDto(productId, quantity))
            if (response.isSuccessful) response.body()!!.toDomain()
            else error("Error ${response.code()}: ${response.errorBody()?.string()}")
        }

    override suspend fun confirmOrder(orderId: Int): Result<Order> = runCatching {
        val response = api.confirmOrder(orderId)
        if (response.isSuccessful) response.body()!!.toDomain()
        else error("Error ${response.code()}: ${response.errorBody()?.string()}")
    }

    override suspend fun updateStatus(orderId: Int, status: OrderStatus): Result<Order> =
        runCatching {
            val response = api.updateStatus(orderId, UpdateStatusRequestDto(status.value))
            if (response.isSuccessful) response.body()!!.toDomain()
            else error("Error ${response.code()}")
        }

    override suspend fun getStats(): Result<Map<String, Any>> = runCatching {
        val response = api.getStats()
        if (response.isSuccessful) {
            val s = response.body()!!
            mapOf(
                "total_orders"  to s.totalOrders,
                "total_revenue" to s.totalRevenue,
                "by_status"     to s.byStatus,
            )
        } else error("Error ${response.code()}")
    }
}
```

### Actualizar `di/RepositoryModule.kt`

```kotlin
// di/RepositoryModule.kt — agregar binding
@Binds @Singleton
abstract fun bindOrderRepository(impl: OrderRepositoryImpl): OrderRepository
```

---

## 5.2 Pantalla de detalle — `presentation/ui/public/product/ProductDetailScreen.kt`

```kotlin
// presentation/ui/public/product/ProductDetailScreen.kt
package com.shopapp.presentation.ui.public.product

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import coil.compose.AsyncImage
import com.shopapp.domain.model.Product
import com.shopapp.domain.repository.ProductRepository
import com.shopapp.presentation.components.*
import com.shopapp.presentation.viewmodel.CartViewModel
import com.shopapp.theme.*
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

// ── ViewModel de detalle ──────────────────────────────────────

sealed interface ProductDetailUiState {
    data object Loading                     : ProductDetailUiState
    data class  Success(val product: Product) : ProductDetailUiState
    data class  Error(val message: String)  : ProductDetailUiState
}

@HiltViewModel
class ProductDetailViewModel @Inject constructor(
    private val repository: ProductRepository,
) : ViewModel() {

    private val _state = MutableStateFlow<ProductDetailUiState>(ProductDetailUiState.Loading)
    val state: StateFlow<ProductDetailUiState> = _state.asStateFlow()

    fun load(id: Int) {
        viewModelScope.launch {
            _state.value = ProductDetailUiState.Loading
            repository.getProduct(id)
                .onSuccess { _state.value = ProductDetailUiState.Success(it) }
                .onFailure { _state.value = ProductDetailUiState.Error(it.message ?: "Error") }
        }
    }
}

// ── Screen ────────────────────────────────────────────────────

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ProductDetailScreen(
    productId:   Int,
    onBack:      () -> Unit,
    cartViewModel: CartViewModel,
    viewModel:   ProductDetailViewModel = hiltViewModel(),
) {
    val state by viewModel.state.collectAsState()

    LaunchedEffect(productId) { viewModel.load(productId) }

    when (val s = state) {
        is ProductDetailUiState.Loading -> LoadingScreen("Cargando producto...")
        is ProductDetailUiState.Error   -> ErrorScreen(s.message, onRetry = { viewModel.load(productId) })
        is ProductDetailUiState.Success -> ProductDetailContent(
            product       = s.product,
            onBack        = onBack,
            cartViewModel = cartViewModel,
        )
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
private fun ProductDetailContent(
    product:       Product,
    onBack:        () -> Unit,
    cartViewModel: CartViewModel,
) {
    var quantity by remember { mutableIntStateOf(1) }
    var added    by remember { mutableStateOf(false) }

    val subtotal = product.price * quantity

    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(Background)
            .verticalScroll(rememberScrollState()),
    ) {
        // ── Imagen con TopBar flotante ─────────────────────────
        Box(modifier = Modifier.fillMaxWidth().height(320.dp)) {
            if (product.imageUrl != null) {
                AsyncImage(
                    model              = product.imageUrl,
                    contentDescription = product.name,
                    contentScale       = ContentScale.Crop,
                    modifier           = Modifier.fillMaxSize(),
                )
            } else {
                Box(
                    modifier         = Modifier.fillMaxSize().background(Surface2),
                    contentAlignment = Alignment.Center,
                ) { Text("📦", fontSize = 72.sp) }
            }

            // Sin stock banner
            if (!product.inStock) {
                Box(
                    modifier         = Modifier
                        .fillMaxWidth()
                        .align(Alignment.BottomCenter)
                        .background(Error.copy(alpha = 0.9f))
                        .padding(12.dp),
                    contentAlignment = Alignment.Center,
                ) {
                    Text(
                        "PRODUCTO AGOTADO",
                        color      = MaterialTheme.colorScheme.onError,
                        fontWeight = FontWeight.Bold,
                        style      = MaterialTheme.typography.labelLarge,
                    )
                }
            }

            // Botón volver
            IconButton(
                onClick  = onBack,
                modifier = Modifier
                    .align(Alignment.TopStart)
                    .padding(8.dp)
                    .background(Background.copy(alpha = 0.7f), RoundedCornerShape(50)),
            ) {
                Icon(Icons.Default.ArrowBack, contentDescription = "Volver", tint = TextPrimary)
            }
        }

        // ── Info ──────────────────────────────────────────────
        Column(modifier = Modifier.padding(24.dp)) {

            // Categoría
            product.categoryName?.let {
                Text(
                    text       = it.uppercase(),
                    style      = MaterialTheme.typography.labelSmall,
                    color      = Accent,
                    letterSpacing = 1.sp,
                )
                Spacer(Modifier.height(6.dp))
            }

            // Nombre
            Text(
                text       = product.name,
                style      = MaterialTheme.typography.headlineLarge,
                fontWeight = FontWeight.Bold,
                color      = TextPrimary,
            )
            Spacer(Modifier.height(12.dp))

            // Precios
            Text(
                text       = "$${"%.2f".format(product.price)}",
                fontSize   = 32.sp,
                fontWeight = FontWeight.Bold,
                color      = Accent,
            )
            Text(
                text  = "$${"%.2f".format(product.priceWithTax)} con IVA (15%)",
                style = MaterialTheme.typography.bodySmall,
                color = TextSecondary,
            )
            Spacer(Modifier.height(16.dp))

            // Stock
            Row(verticalAlignment = Alignment.CenterVertically) {
                Box(
                    modifier = Modifier
                        .size(8.dp)
                        .background(
                            if (product.inStock) Success else Error,
                            RoundedCornerShape(50),
                        ),
                )
                Spacer(Modifier.width(8.dp))
                Text(
                    text  = if (product.inStock) "${product.stock} unidades disponibles"
                            else "Producto agotado",
                    color = if (product.inStock) TextSecondary else Error,
                    style = MaterialTheme.typography.bodySmall,
                )
            }
            Spacer(Modifier.height(16.dp))

            // Descripción
            if (product.description.isNotBlank()) {
                HorizontalDivider(color = Border, thickness = 0.5.dp)
                Spacer(Modifier.height(16.dp))
                Text(
                    text  = product.description,
                    style = MaterialTheme.typography.bodyMedium,
                    color = TextSecondary,
                )
                Spacer(Modifier.height(16.dp))
            }

            // Selector de cantidad
            if (product.inStock) {
                HorizontalDivider(color = Border, thickness = 0.5.dp)
                Spacer(Modifier.height(16.dp))

                Row(
                    modifier          = Modifier.fillMaxWidth(),
                    verticalAlignment = Alignment.CenterVertically,
                    horizontalArrangement = Arrangement.SpaceBetween,
                ) {
                    Text(
                        text       = "Cantidad",
                        style      = MaterialTheme.typography.titleMedium,
                        fontWeight = FontWeight.SemiBold,
                        color      = TextPrimary,
                    )
                    Row(verticalAlignment = Alignment.CenterVertically) {
                        IconButton(
                            onClick  = { if (quantity > 1) quantity-- },
                            enabled  = quantity > 1,
                        ) {
                            Icon(
                                Icons.Default.Remove,
                                contentDescription = "Menos",
                                tint = if (quantity > 1) TextPrimary else TextFaint,
                            )
                        }
                        Text(
                            text       = quantity.toString(),
                            style      = MaterialTheme.typography.titleLarge,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                            modifier   = Modifier.padding(horizontal = 16.dp),
                        )
                        IconButton(
                            onClick = { if (quantity < product.stock) quantity++ },
                            enabled = quantity < product.stock,
                        ) {
                            Icon(
                                Icons.Default.Add,
                                contentDescription = "Más",
                                tint = if (quantity < product.stock) TextPrimary else TextFaint,
                            )
                        }
                    }
                }

                // Subtotal
                Spacer(Modifier.height(12.dp))
                Surface(
                    color  = Surface2,
                    shape  = MaterialTheme.shapes.medium,
                    modifier = Modifier.fillMaxWidth(),
                ) {
                    Row(
                        modifier              = Modifier.padding(16.dp),
                        horizontalArrangement = Arrangement.SpaceBetween,
                        verticalAlignment     = Alignment.CenterVertically,
                    ) {
                        Text("Subtotal", color = TextSecondary, style = MaterialTheme.typography.bodyMedium)
                        Text(
                            text       = "$${"%.2f".format(subtotal)}",
                            style      = MaterialTheme.typography.titleMedium,
                            fontWeight = FontWeight.Bold,
                            color      = Accent,
                        )
                    }
                }
                Spacer(Modifier.height(20.dp))

                // Botón añadir al carrito
                Button(
                    onClick = {
                        cartViewModel.addItem(product, quantity)
                        added = true
                    },
                    enabled  = !added,
                    modifier = Modifier.fillMaxWidth().height(52.dp),
                    colors   = ButtonDefaults.buttonColors(
                        containerColor         = if (added) Success else Accent,
                        contentColor           = AccentOnDark,
                        disabledContainerColor = Success,
                        disabledContentColor   = AccentOnDark,
                    ),
                    shape    = MaterialTheme.shapes.medium,
                ) {
                    Icon(
                        imageVector        = if (added) Icons.Default.Check else Icons.Default.ShoppingCart,
                        contentDescription = null,
                        modifier           = Modifier.size(18.dp),
                    )
                    Spacer(Modifier.width(8.dp))
                    Text(
                        text       = if (added) "¡Añadido al carrito!" else
                            "Añadir${if (quantity > 1) " $quantity×" else ""} al carrito",
                        fontWeight = FontWeight.Bold,
                        style      = MaterialTheme.typography.labelLarge,
                    )
                }

                // Resetear el estado "añadido"
                LaunchedEffect(added) {
                    if (added) {
                        kotlinx.coroutines.delay(2_000)
                        added = false
                    }
                }
            } else {
                Spacer(Modifier.height(20.dp))
                Button(
                    onClick  = {},
                    enabled  = false,
                    modifier = Modifier.fillMaxWidth().height(52.dp),
                    colors   = ButtonDefaults.buttonColors(
                        disabledContainerColor = Surface2,
                        disabledContentColor   = TextFaint,
                    ),
                    shape = MaterialTheme.shapes.medium,
                ) {
                    Text("Producto agotado", fontWeight = FontWeight.Bold)
                }
            }
        }
    }
}
```

---

## 5.3 CartViewModel — actualizar con checkout

```kotlin
// presentation/viewmodel/CartViewModel.kt — añadir checkout
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.Product
import com.shopapp.domain.repository.AuthRepository
import com.shopapp.domain.repository.OrderRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

data class CartItem(
    val product:  Product,
    val quantity: Int,
)

sealed interface CheckoutState {
    data object Idle                          : CheckoutState
    data object Loading                       : CheckoutState
    data class  Success(val orderId: Int)     : CheckoutState
    data class  Error(val message: String)   : CheckoutState
}

@HiltViewModel
class CartViewModel @Inject constructor(
    private val orderRepository: OrderRepository,
) : ViewModel() {

    private val _items         = MutableStateFlow<List<CartItem>>(emptyList())
    val items: StateFlow<List<CartItem>> = _items.asStateFlow()

    val totalItems: StateFlow<Int> = _items
        .map { it.sumOf { i -> i.quantity } }
        .stateIn(viewModelScope, SharingStarted.Eagerly, 0)

    val subtotal: StateFlow<Double> = _items
        .map { it.sumOf { i -> i.product.price * i.quantity } }
        .stateIn(viewModelScope, SharingStarted.Eagerly, 0.0)

    val totalWithTax: StateFlow<Double> = _items
        .map { it.sumOf { i -> i.product.priceWithTax * i.quantity } }
        .stateIn(viewModelScope, SharingStarted.Eagerly, 0.0)

    private val _checkoutState = MutableStateFlow<CheckoutState>(CheckoutState.Idle)
    val checkoutState: StateFlow<CheckoutState> = _checkoutState.asStateFlow()

    // ── CRUD del carrito ──────────────────────────────────────

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
        _items.update { it.filter { i -> i.product.id != productId } }
    }

    fun clearCart() { _items.value = emptyList() }

    fun resetCheckout() { _checkoutState.value = CheckoutState.Idle }

    // ── Checkout — 3 pasos ────────────────────────────────────

    fun checkout() {
        val currentItems = _items.value
        if (currentItems.isEmpty()) {
            _checkoutState.value = CheckoutState.Error("El carrito está vacío")
            return
        }
        viewModelScope.launch {
            _checkoutState.value = CheckoutState.Loading

            // 1. Crear pedido vacío
            val order = orderRepository.createOrder().getOrElse {
                _checkoutState.value = CheckoutState.Error(it.message ?: "Error al crear pedido")
                return@launch
            }

            // 2. Añadir cada ítem
            for (item in currentItems) {
                orderRepository.addItem(order.id, item.product.id, item.quantity).getOrElse {
                    _checkoutState.value = CheckoutState.Error("Error al añadir ${item.product.name}")
                    return@launch
                }
            }

            // 3. Confirmar
            val confirmed = orderRepository.confirmOrder(order.id).getOrElse {
                _checkoutState.value = CheckoutState.Error(it.message ?: "Error al confirmar")
                return@launch
            }

            clearCart()
            _checkoutState.value = CheckoutState.Success(confirmed.id)
        }
    }
}
```

---

## 5.4 CartBottomSheet — `presentation/ui/public/cart/CartBottomSheet.kt`

```kotlin
// presentation/ui/public/cart/CartBottomSheet.kt
package com.shopapp.presentation.ui.public.cart

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import coil.compose.AsyncImage
import com.shopapp.presentation.viewmodel.CartItem
import com.shopapp.presentation.viewmodel.CartViewModel
import com.shopapp.presentation.viewmodel.CheckoutState
import com.shopapp.theme.*

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun CartBottomSheet(
    cartViewModel:   CartViewModel,
    isAuthenticated: Boolean,
    onDismiss:       () -> Unit,
    onLoginRequired: () -> Unit,
    onOrderSuccess:  (Int) -> Unit,
) {
    val items         by cartViewModel.items.collectAsState()
    val subtotal      by cartViewModel.subtotal.collectAsState()
    val totalWithTax  by cartViewModel.totalWithTax.collectAsState()
    val checkoutState by cartViewModel.checkoutState.collectAsState()

    // Navegar al éxito cuando el checkout termina
    LaunchedEffect(checkoutState) {
        if (checkoutState is CheckoutState.Success) {
            onOrderSuccess((checkoutState as CheckoutState.Success).orderId)
            cartViewModel.resetCheckout()
        }
    }

    ModalBottomSheet(
        onDismissRequest = onDismiss,
        containerColor   = Surface,
        dragHandle       = {
            Box(
                modifier         = Modifier
                    .padding(vertical = 12.dp)
                    .size(40.dp, 4.dp)
                    .background(Border, RoundedCornerShape(2.dp)),
            )
        },
    ) {
        Column(
            modifier = Modifier
                .fillMaxWidth()
                .navigationBarsPadding()
                .padding(bottom = 16.dp),
        ) {
            // ── Header ────────────────────────────────────────
            Row(
                modifier              = Modifier
                    .fillMaxWidth()
                    .padding(horizontal = 24.dp),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.CenterVertically,
            ) {
                Column {
                    Text(
                        text       = "Mi carrito",
                        style      = MaterialTheme.typography.titleLarge,
                        fontWeight = FontWeight.Bold,
                        color      = TextPrimary,
                    )
                    if (items.isNotEmpty()) {
                        Text(
                            text  = "${items.sumOf { it.quantity }} producto${if (items.sumOf { it.quantity } != 1) "s" else ""}",
                            style = MaterialTheme.typography.bodySmall,
                            color = TextSecondary,
                        )
                    }
                }
                if (items.isNotEmpty()) {
                    IconButton(onClick = cartViewModel::clearCart) {
                        Icon(Icons.Default.DeleteOutline, contentDescription = "Vaciar", tint = Error)
                    }
                }
            }

            Spacer(Modifier.height(16.dp))
            HorizontalDivider(color = Border, thickness = 0.5.dp)

            // ── Estado vacío ──────────────────────────────────
            if (items.isEmpty() && checkoutState !is CheckoutState.Success) {
                Column(
                    modifier            = Modifier
                        .fillMaxWidth()
                        .padding(48.dp),
                    horizontalAlignment = Alignment.CenterHorizontally,
                ) {
                    Text("🛒", fontSize = 52.sp)
                    Spacer(Modifier.height(12.dp))
                    Text(
                        "Tu carrito está vacío",
                        style      = MaterialTheme.typography.titleMedium,
                        fontWeight = FontWeight.Bold,
                        color      = TextPrimary,
                    )
                    Text(
                        "Añade productos desde el catálogo",
                        style = MaterialTheme.typography.bodySmall,
                        color = TextSecondary,
                    )
                }
            }

            // ── Lista de ítems ────────────────────────────────
            if (items.isNotEmpty()) {
                LazyColumn(
                    modifier          = Modifier
                        .fillMaxWidth()
                        .heightIn(max = 320.dp),
                    contentPadding    = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
                    verticalArrangement = Arrangement.spacedBy(8.dp),
                ) {
                    items(items, key = { it.product.id }) { item ->
                        CartItemRow(
                            item          = item,
                            onIncrease    = { cartViewModel.updateQuantity(item.product.id, item.quantity + 1) },
                            onDecrease    = { cartViewModel.updateQuantity(item.product.id, item.quantity - 1) },
                            onRemove      = { cartViewModel.removeItem(item.product.id) },
                        )
                    }
                }

                HorizontalDivider(color = Border, thickness = 0.5.dp)

                // Totales
                Column(modifier = Modifier.padding(horizontal = 24.dp, vertical = 16.dp)) {
                    TotalRow("Subtotal", "$${"%.2f".format(subtotal)}", false)
                    Spacer(Modifier.height(4.dp))
                    TotalRow("IVA (15%)", "$${"%.2f".format(totalWithTax - subtotal)}", false)
                    Spacer(Modifier.height(8.dp))
                    HorizontalDivider(color = Border, thickness = 0.5.dp)
                    Spacer(Modifier.height(8.dp))
                    TotalRow("Total", "$${"%.2f".format(totalWithTax)}", true)
                }

                // Error de checkout
                if (checkoutState is CheckoutState.Error) {
                    Surface(
                        color    = Error.copy(alpha = 0.1f),
                        shape    = MaterialTheme.shapes.small,
                        modifier = Modifier
                            .fillMaxWidth()
                            .padding(horizontal = 24.dp),
                    ) {
                        Text(
                            text     = (checkoutState as CheckoutState.Error).message,
                            color    = Error,
                            style    = MaterialTheme.typography.bodySmall,
                            modifier = Modifier.padding(12.dp),
                        )
                    }
                    Spacer(Modifier.height(8.dp))
                }

                // Auth notice
                if (!isAuthenticated) {
                    Surface(
                        color    = Accent.copy(alpha = 0.08f),
                        shape    = MaterialTheme.shapes.small,
                        modifier = Modifier
                            .fillMaxWidth()
                            .padding(horizontal = 24.dp),
                    ) {
                        Text(
                            text     = "💡 Inicia sesión para confirmar el pedido",
                            color    = Accent,
                            style    = MaterialTheme.typography.bodySmall,
                            modifier = Modifier.padding(12.dp),
                        )
                    }
                    Spacer(Modifier.height(12.dp))
                }

                // Botón checkout
                val isLoading = checkoutState is CheckoutState.Loading
                Button(
                    onClick = {
                        if (!isAuthenticated) onLoginRequired()
                        else cartViewModel.checkout()
                    },
                    enabled  = !isLoading,
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(52.dp)
                        .padding(horizontal = 24.dp),
                    colors   = ButtonDefaults.buttonColors(
                        containerColor         = Accent,
                        contentColor           = AccentOnDark,
                        disabledContainerColor = Accent.copy(alpha = 0.5f),
                    ),
                    shape    = MaterialTheme.shapes.medium,
                ) {
                    if (isLoading) {
                        CircularProgressIndicator(
                            color       = AccentOnDark,
                            modifier    = Modifier.size(18.dp),
                            strokeWidth = 2.dp,
                        )
                        Spacer(Modifier.width(8.dp))
                        Text("Procesando...", fontWeight = FontWeight.Bold)
                    } else if (!isAuthenticated) {
                        Text("Iniciar sesión para comprar", fontWeight = FontWeight.Bold)
                    } else {
                        Text(
                            "Confirmar — $${"%.2f".format(totalWithTax)}",
                            fontWeight = FontWeight.Bold,
                        )
                    }
                }
            }
        }
    }
}

@Composable
private fun CartItemRow(
    item:       CartItem,
    onIncrease: () -> Unit,
    onDecrease: () -> Unit,
    onRemove:   () -> Unit,
) {
    Surface(
        color  = Surface2,
        shape  = MaterialTheme.shapes.medium,
        modifier = Modifier.fillMaxWidth(),
    ) {
        Row(
            modifier          = Modifier.padding(10.dp),
            verticalAlignment = Alignment.CenterVertically,
        ) {
            // Imagen
            Box(
                modifier         = Modifier
                    .size(58.dp)
                    .background(Surface, RoundedCornerShape(10.dp)),
                contentAlignment = Alignment.Center,
            ) {
                if (item.product.imageUrl != null) {
                    AsyncImage(
                        model              = item.product.imageUrl,
                        contentDescription = item.product.name,
                        contentScale       = ContentScale.Crop,
                        modifier           = Modifier.fillMaxSize(),
                    )
                } else {
                    Text("📦", fontSize = 24.sp)
                }
            }

            Spacer(Modifier.width(12.dp))

            // Info
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text     = item.product.name,
                    style    = MaterialTheme.typography.bodyMedium,
                    fontWeight = FontWeight.SemiBold,
                    color    = TextPrimary,
                    maxLines = 2,
                )
                Text(
                    text  = "$${"%.2f".format(item.product.price)} / ud.",
                    style = MaterialTheme.typography.bodySmall,
                    color = TextSecondary,
                )
            }

            Spacer(Modifier.width(8.dp))

            // Controles de cantidad
            Column(horizontalAlignment = Alignment.CenterHorizontally) {
                Row(verticalAlignment = Alignment.CenterVertically) {
                    IconButton(onClick = onDecrease, modifier = Modifier.size(28.dp)) {
                        Icon(Icons.Default.Remove, null, tint = TextSecondary, modifier = Modifier.size(14.dp))
                    }
                    Text(
                        text       = item.quantity.toString(),
                        fontWeight = FontWeight.Bold,
                        color      = TextPrimary,
                        modifier   = Modifier.padding(horizontal = 8.dp),
                    )
                    IconButton(
                        onClick  = onIncrease,
                        enabled  = item.quantity < item.product.stock,
                        modifier = Modifier.size(28.dp),
                    ) {
                        Icon(
                            Icons.Default.Add, null,
                            tint     = if (item.quantity < item.product.stock) TextSecondary else TextFaint,
                            modifier = Modifier.size(14.dp),
                        )
                    }
                }
                Text(
                    text       = "$${"%.2f".format(item.product.price * item.quantity)}",
                    style      = MaterialTheme.typography.bodySmall,
                    fontWeight = FontWeight.Bold,
                    color      = Accent,
                )
            }

            IconButton(onClick = onRemove, modifier = Modifier.size(32.dp)) {
                Icon(Icons.Default.Close, contentDescription = "Quitar", tint = TextFaint, modifier = Modifier.size(16.dp))
            }
        }
    }
}

@Composable
private fun TotalRow(label: String, value: String, isFinal: Boolean) {
    Row(
        modifier              = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment     = Alignment.CenterVertically,
    ) {
        Text(
            text       = label,
            style      = if (isFinal) MaterialTheme.typography.titleMedium else MaterialTheme.typography.bodyMedium,
            fontWeight = if (isFinal) FontWeight.Bold else FontWeight.Normal,
            color      = if (isFinal) TextPrimary else TextSecondary,
        )
        Text(
            text       = value,
            style      = if (isFinal) MaterialTheme.typography.titleMedium else MaterialTheme.typography.bodyMedium,
            fontWeight = if (isFinal) FontWeight.ExtraBold else FontWeight.SemiBold,
            color      = if (isFinal) Accent else TextPrimary,
        )
    }
}
```

---

## 5.5 Actualizar `NavGraph.kt` — integrar pantallas del M5

```kotlin
// presentation/navigation/NavGraph.kt — reemplazar los placeholders del M5

// Importar:
import com.shopapp.presentation.ui.public.product.ProductDetailScreen
import com.shopapp.presentation.ui.public.cart.CartBottomSheet

// Dentro del NavHost — reemplazar los composable de product y cart:

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

// ── Cart BottomSheet (manejado con estado local) ──────────────
// En el Scaffold — añadir variable de estado y el BottomSheet:
```

Actualizar el Scaffold en `NavGraph.kt`:

```kotlin
// NavGraph.kt — dentro de @Composable NavGraph()
var showCart by remember { mutableStateOf(false) }
var confirmedOrderId by remember { mutableStateOf<Int?>(null) }

// El Scaffold ya existe, añadir el BottomSheet al body:
if (showCart) {
    CartBottomSheet(
        cartViewModel   = cartViewModel,
        isAuthenticated = isAuthenticated,
        onDismiss       = { showCart = false },
        onLoginRequired = {
            showCart = false
            navController.navigate(Screen.Login.route)
        },
        onOrderSuccess  = { orderId ->
            confirmedOrderId = orderId
            showCart         = false
        },
    )
}

// Actualizar el BottomNavBar para usar showCart:
BottomNavBar(
    navController = navController,
    cartCount     = cartCount,
    onCartClick   = { showCart = true },   // ya no navega, abre el sheet
)
```

Archivo completo

```kotlin
// NavGraph.kt — dentro de @Composable NavGraph()
// presentation/navigation/NavGraph.kt
package com.shopapp.presentation.navigation

import androidx.compose.foundation.layout.Arrangement
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
import com.shopapp.presentation.ui.auth.LoginScreen
import com.shopapp.presentation.ui.auth.RegisterScreen
import com.shopapp.presentation.ui.uipublic.catalog.CatalogScreen
import com.shopapp.presentation.ui.uipublic.home.HomeScreen
import com.shopapp.presentation.ui.uipublic.product.ProductDetailScreen
import com.shopapp.presentation.ui.uipublic.cart.CartBottomSheet
import com.shopapp.presentation.viewmodel.AuthViewModel
import com.shopapp.presentation.viewmodel.CartViewModel
import com.shopapp.theme.Surface

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
                    navController.navigate(Screen.Login.route) {
                        popUpTo(Screen.Home.route)
                    }
                } else {
                    ScreenWithLogout(
                        title = "Mis pedidos — M6",
                        onLogout = {
                            authViewModel.logout()
                            navController.navigate(Screen.Login.route) {
                                popUpTo(0) { inclusive = true }
                            }
                        }
                    )
                }
            }

            // ── PERFIL ─────────────────────────────
            composable(Screen.Profile.route) {
                if (!isAuthenticated) {
                    navController.navigate(Screen.Login.route) {
                        popUpTo(Screen.Home.route)
                    }
                } else {
                    ScreenWithLogout(
                        title = "Mi perfil — M6",
                        onLogout = {
                            authViewModel.logout()
                            navController.navigate(Screen.Login.route) {
                                popUpTo(0) { inclusive = true }
                            }
                        }
                    ) {
                        LoadingScreen("Mi perfil — M6")
                    }
                }
            }

            // ── ADMIN ──────────────────────────────
            composable(Screen.AdminDashboard.route) {
                if (!isStaff) {
                    navController.navigate(Screen.Home.route) {
                        popUpTo(0)
                    }
                } else {
                    ScreenWithLogout(
                        title = "Admin Dashboard — M8",
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

## ✅ Checkpoint Módulo 5

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Tap en producto del catálogo | Abre `ProductDetailScreen` |
| 2 | Imagen, precio e IVA 15% | Datos reales del backend |
| 3 | `+` / `-` selector de cantidad | No supera el stock disponible |
| 4 | "Añadir al carrito" | Botón verde ✓, badge del BottomNav aumenta |
| 5 | Tap en ícono carrito | `CartBottomSheet` sube desde abajo |
| 6 | Controles de cantidad en el sheet | Subtotal recalculado en tiempo real |
| 7 | Desglose IVA en el sheet | Subtotal + IVA (15%) + Total |
| 8 | Checkout sin login | Notice dorado + botón "Iniciar sesión" |
| 9 | Checkout con login | Spinner "Procesando...", luego cierra el sheet |
| 10 | Verificar en Django admin | Pedido con status "confirmed" e ítems |

```bash
# Verificar en Django
uv run python manage.py shell -c "
from store.models import Order
for o in Order.objects.order_by('-created_at')[:3]:
    print(f'#{o.id} {o.status} \${o.total} — {o.items.count()} items')
"
```

---

## Resumen

| Elemento | Estado |
|---|---|
| `OrderRepositoryImpl` con flujo de 3 pasos | ✅ |
| `ProductDetailScreen` con ViewModel propio | ✅ |
| Selector de cantidad con límite de stock | ✅ |
| Botón "añadido" verde durante 2 segundos | ✅ |
| `CartViewModel` con `checkout()` completo | ✅ |
| `CartBottomSheet` con `ModalBottomSheet` M3 | ✅ |
| Lista de ítems con controles inline | ✅ |
| Desglose subtotal + IVA 15% + total | ✅ |
| Estado `CheckoutState` sealed interface | ✅ |
| Guard de autenticación en el checkout | ✅ |

**Siguiente módulo →** M6: Mis pedidos — historial paginado, detalle con barra de progreso y perfil