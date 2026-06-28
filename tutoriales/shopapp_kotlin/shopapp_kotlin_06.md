# ShopApp Android — Módulo 6
## Mis pedidos — Historial paginado, detalle con barra de progreso y perfil

> **Objetivo:** Zona privada del cliente completa: historial de pedidos con filtros por estado, pantalla de detalle con barra de progreso visual y pantalla de perfil.
> **Checkpoint final:** ver el pedido creado en M5 con estado "confirmed", navegar al detalle con la barra de progreso iluminada y ver el perfil del usuario logueado.

---

## 6.1 StatusBadge — `presentation/components/StatusBadge.kt`

```kotlin
// presentation/components/StatusBadge.kt
package com.shopapp.presentation.components

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.shopapp.domain.model.OrderStatus
import com.shopapp.theme.*

fun orderStatusColor(status: OrderStatus): Color = when (status) {
    OrderStatus.PENDING   -> StatusPending
    OrderStatus.CONFIRMED -> StatusConfirmed
    OrderStatus.SHIPPED   -> StatusShipped
    OrderStatus.DELIVERED -> StatusDelivered
    OrderStatus.CANCELLED -> StatusCancelled
}

@Composable
fun StatusBadge(status: OrderStatus, modifier: Modifier = Modifier) {
    val color = orderStatusColor(status)
    Row(
        modifier          = modifier
            .background(color.copy(alpha = 0.12f), RoundedCornerShape(999.dp))
            .padding(horizontal = 10.dp, vertical = 4.dp),
        verticalAlignment = Alignment.CenterVertically,
        horizontalArrangement = Arrangement.spacedBy(6.dp),
    ) {
        Box(
            modifier = Modifier
                .size(6.dp)
                .background(color, RoundedCornerShape(50)),
        )
        Text(
            text       = status.label,
            color      = color,
            fontSize   = 11.sp,
            fontWeight = FontWeight.Bold,
            letterSpacing = 0.3.sp,
        )
    }
}
```

---

## 6.2 ViewModel de pedidos — `presentation/viewmodel/OrdersViewModel.kt`

```kotlin
// presentation/viewmodel/OrdersViewModel.kt
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.Order
import com.shopapp.domain.model.OrderStatus
import com.shopapp.domain.repository.OrderRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

data class OrdersUiState(
    val orders:       List<Order> = emptyList(),
    val isLoading:    Boolean     = false,
    val isLoadingMore:Boolean     = false,
    val error:        String?     = null,
    val total:        Int         = 0,
    val hasMore:      Boolean     = false,
    val statusFilter: String      = "",
    val page:         Int         = 1,
)

@HiltViewModel
class OrdersViewModel @Inject constructor(
    private val repository: OrderRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(OrdersUiState())
    val state: StateFlow<OrdersUiState> = _state.asStateFlow()

    init { load() }

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
            repository.getOrders(
                page   = page,
                status = current.statusFilter.ifBlank { null },
            ).onSuccess { (orders, total) ->
                _state.update { s ->
                    s.copy(
                        orders        = if (reset) orders else s.orders + orders,
                        total         = total,
                        hasMore       = (if (reset) orders else s.orders + orders).size < total,
                        isLoading     = false,
                        isLoadingMore = false,
                        page          = page + 1,
                        error         = null,
                    )
                }
            }.onFailure { e ->
                _state.update { it.copy(isLoading = false, isLoadingMore = false, error = e.message) }
            }
        }
    }

    fun setStatusFilter(status: String) {
        _state.update { it.copy(statusFilter = status) }
        load(reset = true)
    }

    fun loadMore() = load(reset = false)
    fun refresh()  = load(reset = true)
}
```

---

## 6.3 ViewModel de detalle — `presentation/viewmodel/OrderDetailViewModel.kt`

```kotlin
// presentation/viewmodel/OrderDetailViewModel.kt
package com.shopapp.presentation.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.shopapp.domain.model.Order
import com.shopapp.domain.repository.OrderRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch
import javax.inject.Inject

sealed interface OrderDetailUiState {
    data object Loading                    : OrderDetailUiState
    data class  Success(val order: Order)  : OrderDetailUiState
    data class  Error(val message: String) : OrderDetailUiState
}

@HiltViewModel
class OrderDetailViewModel @Inject constructor(
    private val repository: OrderRepository,
) : ViewModel() {

    private val _state = MutableStateFlow<OrderDetailUiState>(OrderDetailUiState.Loading)
    val state: StateFlow<OrderDetailUiState> = _state.asStateFlow()

    fun load(id: Int) {
        viewModelScope.launch {
            _state.value = OrderDetailUiState.Loading
            repository.getOrder(id)
                .onSuccess { _state.value = OrderDetailUiState.Success(it) }
                .onFailure { _state.value = OrderDetailUiState.Error(it.message ?: "Error") }
        }
    }
}
```

---

## 6.4 Pantalla historial de pedidos — `presentation/ui/client/orders/OrdersScreen.kt`

```kotlin
// presentation/ui/client/orders/OrdersScreen.kt
package com.shopapp.presentation.ui.client.orders

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.lazy.rememberLazyListState
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Refresh
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.domain.model.Order
import com.shopapp.domain.model.OrderStatus
import com.shopapp.presentation.components.StatusBadge
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.components.ErrorScreen
import com.shopapp.presentation.viewmodel.OrdersViewModel
import com.shopapp.theme.*
import java.text.SimpleDateFormat
import java.util.Locale

private val STATUS_FILTERS = listOf(
    "" to "Todos",
    OrderStatus.PENDING.value   to "Pendientes",
    OrderStatus.CONFIRMED.value to "Confirmados",
    OrderStatus.SHIPPED.value   to "Enviados",
    OrderStatus.DELIVERED.value to "Entregados",
    OrderStatus.CANCELLED.value to "Cancelados",
)

@Composable
fun OrdersScreen(
    onOrderClick: (Int) -> Unit,
    viewModel:    OrdersViewModel = hiltViewModel(),
) {
    val state     by viewModel.state.collectAsState()
    val listState  = rememberLazyListState()

    // Cargar más al llegar al final
    val shouldLoadMore by remember {
        derivedStateOf {
            val lastVisible = listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
            val total       = listState.layoutInfo.totalItemsCount
            lastVisible >= total - 2 && !state.isLoadingMore && state.hasMore
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
                            text       = "Mis pedidos",
                            style      = MaterialTheme.typography.headlineMedium,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                        )
                        Text(
                            text  = if (state.isLoading) "..." else "${state.total} pedidos",
                            style = MaterialTheme.typography.bodySmall,
                            color = TextSecondary,
                        )
                    }
                    IconButton(onClick = viewModel::refresh) {
                        Icon(Icons.Default.Refresh, contentDescription = "Actualizar", tint = TextSecondary)
                    }
                }

                Spacer(Modifier.height(12.dp))

                // Filtros por estado
                LazyRow(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                    items(STATUS_FILTERS) { (value, label) ->
                        FilterChip(
                            selected = state.statusFilter == value,
                            onClick  = { viewModel.setStatusFilter(value) },
                            label    = { Text(label, style = MaterialTheme.typography.labelSmall) },
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
            state.isLoading && state.orders.isEmpty() ->
                LoadingScreen("Cargando pedidos...")

            state.error != null && state.orders.isEmpty() ->
                ErrorScreen(state.error!!, onRetry = viewModel::refresh)

            state.orders.isEmpty() -> {
                Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                        Text("📦", fontSize = 52.sp)
                        Spacer(Modifier.height(12.dp))
                        Text(
                            text       = if (state.statusFilter.isBlank()) "Sin pedidos"
                                         else "Sin pedidos con este estado",
                            style      = MaterialTheme.typography.titleMedium,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                        )
                        Text(
                            text  = "Tus pedidos aparecerán aquí",
                            style = MaterialTheme.typography.bodySmall,
                            color = TextSecondary,
                        )
                    }
                }
            }

            else -> {
                LazyColumn(
                    state          = listState,
                    modifier       = Modifier.fillMaxSize(),
                    contentPadding = PaddingValues(16.dp),
                    verticalArrangement = Arrangement.spacedBy(12.dp),
                ) {
                    items(state.orders, key = { it.id }) { order ->
                        OrderCard(order = order, onClick = { onOrderClick(order.id) })
                    }

                    if (state.isLoadingMore) {
                        item {
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

// ── OrderCard ─────────────────────────────────────────────────

@Composable
fun OrderCard(order: Order, onClick: () -> Unit) {
    val inputFmt  = SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSSSS'Z'", Locale.getDefault())
    val outputFmt = SimpleDateFormat("dd MMM yyyy", Locale("es"))
    val dateStr   = runCatching {
        outputFmt.format(inputFmt.parse(order.createdAt)!!)
    }.getOrDefault(order.createdAt.take(10))

    Surface(
        onClick        = onClick,
        shape          = MaterialTheme.shapes.large,
        color          = Surface,
        tonalElevation = 0.dp,
        modifier       = Modifier.fillMaxWidth(),
    ) {
        Column(modifier = Modifier.padding(16.dp)) {

            // Header
            Row(
                modifier              = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.Top,
            ) {
                Column {
                    Text(
                        text       = "Pedido #${order.id}",
                        style      = MaterialTheme.typography.titleMedium,
                        fontWeight = FontWeight.Bold,
                        color      = TextPrimary,
                    )
                    Text(
                        text  = dateStr,
                        style = MaterialTheme.typography.bodySmall,
                        color = TextSecondary,
                    )
                }
                StatusBadge(status = order.status)
            }

            Spacer(Modifier.height(12.dp))

            // Preview ítems como chips
            Row(
                horizontalArrangement = Arrangement.spacedBy(6.dp),
                modifier              = Modifier.fillMaxWidth(),
            ) {
                val preview = order.items.take(3)
                preview.forEach { item ->
                    Surface(
                        color  = Surface2,
                        shape  = MaterialTheme.shapes.extraSmall,
                    ) {
                        Text(
                            text     = "${item.quantity}× ${item.productName}",
                            style    = MaterialTheme.typography.bodySmall.copy(fontSize = 11.sp),
                            color    = TextSecondary,
                            modifier = Modifier.padding(horizontal = 8.dp, vertical = 3.dp),
                            maxLines = 1,
                        )
                    }
                }
                if (order.items.size > 3) {
                    Text(
                        text  = "+${order.items.size - 3} más",
                        style = MaterialTheme.typography.bodySmall,
                        color = TextFaint,
                        modifier = Modifier.align(Alignment.CenterVertically),
                    )
                }
            }

            Spacer(Modifier.height(12.dp))
            HorizontalDivider(color = BorderLight, thickness = 0.5.dp)
            Spacer(Modifier.height(10.dp))

            // Footer
            Row(
                modifier              = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.CenterVertically,
            ) {
                Text(
                    text  = "${order.numItems} producto${if (order.numItems != 1) "s" else ""}",
                    style = MaterialTheme.typography.bodySmall,
                    color = TextSecondary,
                )
                Text(
                    text       = "$${"%.2f".format(order.total)}",
                    style      = MaterialTheme.typography.titleMedium,
                    fontWeight = FontWeight.ExtraBold,
                    color      = Accent,
                )
            }
        }
    }
}
```

---

## 6.5 Pantalla detalle de pedido — `presentation/ui/client/orders/OrderDetailScreen.kt`

```kotlin
// presentation/ui/client/orders/OrderDetailScreen.kt
package com.shopapp.presentation.ui.client.orders

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.ArrowBack
import androidx.compose.material.icons.filled.Package
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.hilt.navigation.compose.hiltViewModel
import com.shopapp.domain.model.Order
import com.shopapp.domain.model.OrderItem
import com.shopapp.domain.model.OrderStatus
import com.shopapp.presentation.components.ErrorScreen
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.components.StatusBadge
import com.shopapp.presentation.components.orderStatusColor
import com.shopapp.presentation.viewmodel.OrderDetailUiState
import com.shopapp.presentation.viewmodel.OrderDetailViewModel
import com.shopapp.theme.*
import java.text.SimpleDateFormat
import java.util.Locale

private val PROGRESS_STEPS = listOf(
    OrderStatus.PENDING,
    OrderStatus.CONFIRMED,
    OrderStatus.SHIPPED,
    OrderStatus.DELIVERED,
)

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun OrderDetailScreen(
    orderId:   Int,
    onBack:    () -> Unit,
    viewModel: OrderDetailViewModel = hiltViewModel(),
) {
    val state by viewModel.state.collectAsState()
    LaunchedEffect(orderId) { viewModel.load(orderId) }

    when (val s = state) {
        is OrderDetailUiState.Loading ->
            LoadingScreen("Cargando pedido...")
        is OrderDetailUiState.Error   ->
            ErrorScreen(s.message, onRetry = { viewModel.load(orderId) })
        is OrderDetailUiState.Success ->
            OrderDetailContent(order = s.order, onBack = onBack)
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
private fun OrderDetailContent(order: Order, onBack: () -> Unit) {
    val inputFmt  = SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSSSS'Z'", Locale.getDefault())
    val outputFmt = SimpleDateFormat("dd MMM yyyy · HH:mm", Locale("es"))
    val dateStr   = runCatching { outputFmt.format(inputFmt.parse(order.createdAt)!!) }
                        .getOrDefault(order.createdAt.take(16))

    val isCancelled = order.status == OrderStatus.CANCELLED
    val currentStep = PROGRESS_STEPS.indexOf(order.status).coerceAtLeast(0)
    val taxAmount   = order.total - order.total / 1.15
    val subtotal    = order.total - taxAmount

    Scaffold(
        topBar = {
            TopAppBar(
                title = {
                    Column {
                        Text(
                            "Pedido #${order.id}",
                            style      = MaterialTheme.typography.titleMedium,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                        )
                        Text(dateStr, style = MaterialTheme.typography.bodySmall, color = TextSecondary)
                    }
                },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.Default.ArrowBack, contentDescription = "Volver", tint = TextPrimary)
                    }
                },
                actions = { StatusBadge(order.status, modifier = Modifier.padding(end = 16.dp)) },
                colors  = TopAppBarDefaults.topAppBarColors(containerColor = Surface),
            )
        },
        containerColor = Background,
    ) { padding ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(padding)
                .verticalScroll(rememberScrollState())
                .padding(16.dp),
            verticalArrangement = Arrangement.spacedBy(16.dp),
        ) {

            // ── Barra de progreso ──────────────────────────────
            if (!isCancelled) {
                OrderProgressBar(
                    steps       = PROGRESS_STEPS,
                    currentStep = currentStep,
                )
            } else {
                Surface(
                    color    = Error.copy(alpha = 0.08f),
                    shape    = MaterialTheme.shapes.medium,
                    modifier = Modifier.fillMaxWidth(),
                ) {
                    Text(
                        text     = "⚠️ Este pedido fue cancelado",
                        color    = Error,
                        fontWeight = FontWeight.SemiBold,
                        style    = MaterialTheme.typography.bodyMedium,
                        modifier = Modifier.padding(16.dp),
                    )
                }
            }

            // ── Ítems ──────────────────────────────────────────
            SectionCard(title = "Productos (${order.numItems})") {
                Column(verticalArrangement = Arrangement.spacedBy(10.dp)) {
                    order.items.forEach { item ->
                        OrderItemRow(item = item)
                    }
                }
            }

            // ── Resumen de totales ─────────────────────────────
            SectionCard(title = "Resumen") {
                Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                    TotalLine("Subtotal (sin IVA)", subtotal,  false)
                    TotalLine("IVA (15%)",           taxAmount, false)
                    HorizontalDivider(color = Border, thickness = 0.5.dp)
                    TotalLine("Total",               order.total, true)
                }
            }

            // Meta info
            Text(
                text  = "Actualizado: $dateStr",
                style = MaterialTheme.typography.bodySmall,
                color = TextFaint,
                modifier = Modifier.align(Alignment.CenterHorizontally),
            )
        }
    }
}

// ── Barra de progreso ─────────────────────────────────────────

@Composable
private fun OrderProgressBar(steps: List<OrderStatus>, currentStep: Int) {
    Surface(
        color    = Surface,
        shape    = MaterialTheme.shapes.large,
        modifier = Modifier.fillMaxWidth(),
    ) {
        Column(modifier = Modifier.padding(20.dp)) {
            Text(
                text       = "Estado del pedido",
                style      = MaterialTheme.typography.labelSmall,
                color      = TextSecondary,
                letterSpacing = 0.8.sp,
                modifier   = Modifier.padding(bottom = 20.dp),
            )
            Row(
                modifier          = Modifier.fillMaxWidth(),
                verticalAlignment = Alignment.CenterVertically,
            ) {
                steps.forEachIndexed { index, step ->
                    val isDone    = index <= currentStep
                    val isCurrent = index == currentStep
                    val color     = if (isDone) Accent else Border

                    // Nodo
                    Column(
                        horizontalAlignment = Alignment.CenterHorizontally,
                    ) {
                        Box(
                            modifier         = Modifier
                                .size(if (isCurrent) 36.dp else 30.dp)
                                .background(
                                    if (isDone) Accent else Surface2,
                                    CircleShape,
                                ),
                            contentAlignment = Alignment.Center,
                        ) {
                            Text(
                                text       = if (isDone) "✓" else "${index + 1}",
                                color      = if (isDone) AccentOnDark else TextFaint,
                                fontSize   = if (isCurrent) 14.sp else 12.sp,
                                fontWeight = FontWeight.Bold,
                            )
                        }
                        Spacer(Modifier.height(6.dp))
                        Text(
                            text       = step.label,
                            style      = MaterialTheme.typography.bodySmall.copy(fontSize = 10.sp),
                            color      = if (isDone) Accent else TextFaint,
                            fontWeight = if (isCurrent) FontWeight.Bold else FontWeight.Normal,
                        )
                    }

                    // Línea conectora
                    if (index < steps.lastIndex) {
                        Box(
                            modifier = Modifier
                                .weight(1f)
                                .height(2.dp)
                                .padding(bottom = 20.dp)
                                .background(if (index < currentStep) Accent else Border),
                        )
                    }
                }
            }
        }
    }
}

// ── Sub-componentes ───────────────────────────────────────────

@Composable
private fun SectionCard(title: String, content: @Composable ColumnScope.() -> Unit) {
    Surface(color = Surface, shape = MaterialTheme.shapes.large, modifier = Modifier.fillMaxWidth()) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text       = title,
                style      = MaterialTheme.typography.labelSmall,
                color      = TextSecondary,
                letterSpacing = 0.8.sp,
                modifier   = Modifier.padding(bottom = 14.dp),
            )
            content()
        }
    }
}

@Composable
private fun OrderItemRow(item: OrderItem) {
    Row(
        modifier              = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.SpaceBetween,
        verticalAlignment     = Alignment.CenterVertically,
    ) {
        Row(
            verticalAlignment = Alignment.CenterVertically,
            modifier          = Modifier.weight(1f),
        ) {
            Box(
                modifier         = Modifier
                    .size(44.dp)
                    .background(Surface2, MaterialTheme.shapes.small),
                contentAlignment = Alignment.Center,
            ) {
                Text("📦", fontSize = 20.sp)
            }
            Spacer(Modifier.width(12.dp))
            Column {
                Text(
                    text       = item.productName,
                    style      = MaterialTheme.typography.bodyMedium,
                    fontWeight = FontWeight.SemiBold,
                    color      = TextPrimary,
                    maxLines   = 2,
                )
                Text(
                    text  = "$${"%.2f".format(item.unitPrice)} × ${item.quantity} ud.",
                    style = MaterialTheme.typography.bodySmall,
                    color = TextSecondary,
                )
            }
        }
        Spacer(Modifier.width(12.dp))
        Text(
            text       = "$${"%.2f".format(item.subtotal)}",
            style      = MaterialTheme.typography.titleSmall,
            fontWeight = FontWeight.Bold,
            color      = Accent,
        )
    }
}

@Composable
private fun TotalLine(label: String, value: Double, isFinal: Boolean) {
    Row(
        modifier              = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.SpaceBetween,
    ) {
        Text(
            text       = label,
            style      = if (isFinal) MaterialTheme.typography.titleMedium
                         else MaterialTheme.typography.bodyMedium,
            fontWeight = if (isFinal) FontWeight.Bold else FontWeight.Normal,
            color      = if (isFinal) TextPrimary else TextSecondary,
        )
        Text(
            text       = "$${"%.2f".format(value)}",
            style      = if (isFinal) MaterialTheme.typography.titleMedium
                         else MaterialTheme.typography.bodyMedium,
            fontWeight = if (isFinal) FontWeight.ExtraBold else FontWeight.SemiBold,
            color      = if (isFinal) Accent else TextPrimary,
        )
    }
}
```

---

## 6.6 Pantalla de perfil — `presentation/ui/client/profile/ProfileScreen.kt`

```kotlin
// presentation/ui/client/profile/ProfileScreen.kt
package com.shopapp.presentation.ui.client.profile

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.shape.CircleShape
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Logout
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.shopapp.presentation.viewmodel.AuthViewModel
import com.shopapp.theme.*

@Composable
fun ProfileScreen(
    authViewModel: AuthViewModel,
    onLogout:      () -> Unit,
) {
    val user by authViewModel.currentUser.collectAsState()

    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(Background)
            .verticalScroll(rememberScrollState())
            .padding(24.dp),
    ) {
        // ── Avatar y nombre ───────────────────────────────────
        Column(
            modifier            = Modifier
                .fillMaxWidth()
                .padding(vertical = 24.dp),
            horizontalAlignment = Alignment.CenterHorizontally,
        ) {
            Box(
                modifier         = Modifier
                    .size(80.dp)
                    .background(
                        brush = androidx.compose.ui.graphics.Brush.linearGradient(
                            listOf(Accent, AccentLight),
                        ),
                        shape = CircleShape,
                    ),
                contentAlignment = Alignment.Center,
            ) {
                Text(
                    text       = user?.username?.firstOrNull()?.uppercaseChar()?.toString() ?: "?",
                    fontSize   = 36.sp,
                    fontWeight = FontWeight.Bold,
                    color      = AccentOnDark,
                )
            }
            Spacer(Modifier.height(16.dp))
            Text(
                text       = user?.username ?: "—",
                style      = MaterialTheme.typography.headlineMedium,
                fontWeight = FontWeight.Bold,
                color      = TextPrimary,
            )
            Text(
                text  = user?.email ?: "—",
                style = MaterialTheme.typography.bodyMedium,
                color = TextSecondary,
            )
            Spacer(Modifier.height(8.dp))
            if (user?.isStaff == true) {
                Surface(
                    color  = Accent.copy(alpha = 0.15f),
                    shape  = MaterialTheme.shapes.extraSmall,
                ) {
                    Text(
                        text       = "Staff",
                        color      = Accent,
                        fontSize   = 11.sp,
                        fontWeight = FontWeight.Bold,
                        modifier   = Modifier.padding(horizontal = 12.dp, vertical = 4.dp),
                        letterSpacing = 0.8.sp,
                    )
                }
            }
        }

        // ── Info del usuario ──────────────────────────────────
        Surface(
            color    = Surface,
            shape    = MaterialTheme.shapes.large,
            modifier = Modifier.fillMaxWidth(),
        ) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text(
                    text      = "Información de la cuenta",
                    style     = MaterialTheme.typography.labelSmall,
                    color     = TextSecondary,
                    letterSpacing = 0.8.sp,
                    modifier  = Modifier.padding(bottom = 12.dp),
                )

                listOf(
                    "ID de usuario" to (user?.id?.toString() ?: "—"),
                    "Usuario"       to (user?.username ?: "—"),
                    "Email"         to (user?.email ?: "—"),
                    "Rol"           to (if (user?.isStaff == true) "Administrador" else "Cliente"),
                ).forEachIndexed { i, (label, value) ->
                    Row(
                        modifier              = Modifier
                            .fillMaxWidth()
                            .padding(vertical = 10.dp),
                        horizontalArrangement = Arrangement.SpaceBetween,
                    ) {
                        Text(
                            text  = label,
                            style = MaterialTheme.typography.bodyMedium,
                            color = TextSecondary,
                        )
                        Text(
                            text       = value,
                            style      = MaterialTheme.typography.bodyMedium,
                            fontWeight = FontWeight.SemiBold,
                            color      = TextPrimary,
                        )
                    }
                    if (i < 3) HorizontalDivider(color = BorderLight, thickness = 0.5.dp)
                }
            }
        }

        Spacer(Modifier.height(24.dp))

        // ── Botón cerrar sesión ───────────────────────────────
        var showConfirm by remember { mutableStateOf(false) }

        OutlinedButton(
            onClick  = { showConfirm = true },
            modifier = Modifier.fillMaxWidth().height(52.dp),
            colors   = ButtonDefaults.outlinedButtonColors(contentColor = Error),
            border   = ButtonDefaults.outlinedButtonBorder.copy(
                brush = androidx.compose.ui.graphics.SolidColor(Error.copy(alpha = 0.5f)),
            ),
            shape    = MaterialTheme.shapes.medium,
        ) {
            Icon(Icons.Default.Logout, contentDescription = null, modifier = Modifier.size(18.dp))
            Spacer(Modifier.width(8.dp))
            Text("Cerrar sesión", fontWeight = FontWeight.SemiBold)
        }

        // Diálogo de confirmación
        if (showConfirm) {
            AlertDialog(
                onDismissRequest = { showConfirm = false },
                title            = { Text("¿Cerrar sesión?", color = TextPrimary) },
                text             = { Text("Tu sesión se cerrará en este dispositivo.", color = TextSecondary) },
                confirmButton    = {
                    TextButton(onClick = {
                        showConfirm = false
                        authViewModel.logout()
                        onLogout()
                    }) {
                        Text("Cerrar sesión", color = Error, fontWeight = FontWeight.Bold)
                    }
                },
                dismissButton    = {
                    TextButton(onClick = { showConfirm = false }) {
                        Text("Cancelar", color = TextSecondary)
                    }
                },
                containerColor   = Surface,
                shape            = MaterialTheme.shapes.large,
            )
        }
    }
}
```

---

## 6.7 Actualizar `NavGraph.kt` — rutas del cliente

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
import com.shopapp.presentation.ui.client.orders.OrdersScreen
import com.shopapp.presentation.ui.client.orders.OrderDetailScreen
import com.shopapp.presentation.ui.client.profile.ProfileScreen
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

## ✅ Checkpoint Módulo 6

```bash
# Asegurarse de que Django está corriendo y tiene datos
uv run python manage.py runserver 0.0.0.0:8000
```

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Tab "Pedidos" | Lista de pedidos del usuario |
| 2 | Pedido del M5 visible | Card con estado "Confirmado" en azul |
| 3 | Chip "Confirmados" | Solo pedidos confirmados |
| 4 | Tap en la card | Navega al detalle |
| 5 | Barra de progreso | Nodo "Confirmado" en dorado, línea iluminada |
| 6 | Pedido cancelado | Banner rojo, sin barra de progreso |
| 7 | Ítems del pedido | Nombre, precio unitario, cantidad, subtotal |
| 8 | Total con IVA desglosado | Subtotal + IVA 15% + Total |
| 9 | Tab "Perfil" | Avatar con inicial, username, email, rol |
| 10 | "Cerrar sesión" | Diálogo de confirmación |
| 11 | Confirmar logout | Navega a Login, tokens eliminados |
| 12 | Reabrir app tras logout | Empieza en LoginScreen |

---

## Resumen

| Elemento | Estado |
|---|---|
| `StatusBadge` con color por estado | ✅ |
| `OrdersViewModel` con paginación y filtros | ✅ |
| `OrderDetailViewModel` | ✅ |
| `OrdersScreen` con chips de filtro | ✅ |
| `OrderCard` con preview de ítems | ✅ |
| `OrderDetailScreen` con `TopAppBar` | ✅ |
| `OrderProgressBar` — barra visual 4 pasos | ✅ |
| Desglose IVA 15% en el detalle | ✅ |
| `ProfileScreen` con avatar degradado | ✅ |
| `AlertDialog` de confirmación de logout | ✅ |
| Rutas `/orders`, `/orders/{id}`, `/profile` | ✅ |

**Siguiente módulo →** M7: Navegación admin — AdminScaffold con Drawer y Dashboard con KPIs