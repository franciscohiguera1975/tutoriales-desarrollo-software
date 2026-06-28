# ShopApp Android — Módulo 10
## Admin CRUD Pedidos — Lista con filtros, cambio de estado inline y detalle

> **Objetivo:** Gestión completa de pedidos desde el panel admin con filtros por estado, selector de estado inline con dropdown, y pantalla de detalle con ítems y desglose de IVA.
> **Checkpoint final:** filtrar pedidos por estado "pending", cambiar a "confirmed" con el dropdown y verificar que el cliente ve el nuevo estado en su historial.

---

## 10.1 ViewModel de pedidos admin — `presentation/viewmodel/OrdersAdminViewModel.kt`

```kotlin
// presentation/viewmodel/OrdersAdminViewModel.kt
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

data class OrdersAdminUiState(
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
class OrdersAdminViewModel @Inject constructor(
    private val repository: OrderRepository,
) : ViewModel() {

    private val _state = MutableStateFlow(OrdersAdminUiState())
    val state: StateFlow<OrdersAdminUiState> = _state.asStateFlow()

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

    // Cambio de estado con actualización optimista
    fun changeStatus(orderId: Int, newStatus: OrderStatus) {
        val prevStatus = _state.value.orders.find { it.id == orderId }?.status ?: return

        // Actualizar optimistamente
        _state.update { s ->
            s.copy(orders = s.orders.map { o ->
                if (o.id == orderId) o.copy(status = newStatus) else o
            })
        }

        viewModelScope.launch {
            repository.updateStatus(orderId, newStatus)
                .onFailure {
                    // Revertir si falla
                    _state.update { s ->
                        s.copy(orders = s.orders.map { o ->
                            if (o.id == orderId) o.copy(status = prevStatus) else o
                        })
                    }
                }
        }
    }
}
```

---

## 10.2 Selector de estado inline — `presentation/ui/admin/orders/StatusDropdown.kt`

```kotlin
// presentation/ui/admin/orders/StatusDropdown.kt
package com.shopapp.presentation.ui.admin.orders

import androidx.compose.foundation.background
import androidx.compose.foundation.border
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.ArrowDropDown
import androidx.compose.material.icons.filled.Check
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.shopapp.domain.model.OrderStatus
import com.shopapp.presentation.components.orderStatusColor
import com.shopapp.theme.*

@Composable
fun StatusDropdown(
    current:   OrderStatus,
    onChange:  (OrderStatus) -> Unit,
    modifier:  Modifier = Modifier,
) {
    var expanded by remember { mutableStateOf(false) }
    val color    = orderStatusColor(current)

    Box(modifier = modifier) {
        // Botón actual
        Surface(
            onClick = { expanded = true },
            shape   = RoundedCornerShape(999.dp),
            color   = color.copy(alpha = 0.12f),
            border  = androidx.compose.foundation.BorderStroke(
                width = 1.dp,
                color = color.copy(alpha = 0.4f),
            ),
        ) {
            Row(
                modifier          = Modifier.padding(horizontal = 10.dp, vertical = 5.dp),
                verticalAlignment = Alignment.CenterVertically,
                horizontalArrangement = Arrangement.spacedBy(5.dp),
            ) {
                Box(
                    modifier = Modifier
                        .size(7.dp)
                        .background(color, RoundedCornerShape(50)),
                )
                Text(
                    text       = current.label,
                    color      = color,
                    fontSize   = 12.sp,
                    fontWeight = FontWeight.Bold,
                )
                Icon(
                    Icons.Default.ArrowDropDown,
                    contentDescription = null,
                    tint     = color,
                    modifier = Modifier.size(14.dp),
                )
            }
        }

        // Dropdown
        DropdownMenu(
            expanded         = expanded,
            onDismissRequest = { expanded = false },
            modifier         = Modifier.background(Surface),
        ) {
            OrderStatus.entries.forEach { status ->
                val statusColor = orderStatusColor(status)
                val isSelected  = status == current

                DropdownMenuItem(
                    text = {
                        Row(
                            verticalAlignment     = Alignment.CenterVertically,
                            horizontalArrangement = Arrangement.spacedBy(8.dp),
                        ) {
                            Box(
                                modifier = Modifier
                                    .size(8.dp)
                                    .background(statusColor, RoundedCornerShape(50)),
                            )
                            Text(
                                text       = status.label,
                                color      = if (isSelected) statusColor else TextSecondary,
                                fontWeight = if (isSelected) FontWeight.Bold else FontWeight.Normal,
                                fontSize   = 13.sp,
                            )
                        }
                    },
                    trailingIcon = if (isSelected) {
                        { Icon(Icons.Default.Check, null, tint = statusColor, modifier = Modifier.size(14.dp)) }
                    } else null,
                    onClick = {
                        expanded = false
                        if (status != current) onChange(status)
                    },
                    colors = MenuDefaults.itemColors(
                        textColor = if (isSelected) statusColor else TextSecondary,
                    ),
                )
            }
        }
    }
}
```

---

## 10.3 Pantalla de pedidos admin — `presentation/ui/admin/orders/OrdersAdminScreen.kt`

```kotlin
// presentation/ui/admin/orders/OrdersAdminScreen.kt
package com.shopapp.presentation.ui.admin.orders

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.LazyRow
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.lazy.rememberLazyListState
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
import com.shopapp.domain.model.Order
import com.shopapp.domain.model.OrderStatus
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.components.ErrorScreen
import com.shopapp.presentation.viewmodel.OrdersAdminViewModel
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
fun OrdersAdminScreen(
    onOrderDetail: (Int) -> Unit,
    viewModel:     OrdersAdminViewModel = hiltViewModel(),
) {
    val state     by viewModel.state.collectAsState()
    val listState  = rememberLazyListState()

    // Paginación automática
    val shouldLoadMore by remember {
        derivedStateOf {
            val lastVisible = listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
            val total       = listState.layoutInfo.totalItemsCount
            lastVisible >= total - 2 && !state.isLoadingMore && state.hasMore
        }
    }
    LaunchedEffect(shouldLoadMore) { if (shouldLoadMore) viewModel.loadMore() }

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
                            "Pedidos",
                            style      = MaterialTheme.typography.headlineMedium,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                        )
                        Text(
                            "${state.total} pedidos",
                            style = MaterialTheme.typography.bodySmall,
                            color = TextSecondary,
                        )
                    }
                    IconButton(onClick = viewModel::refresh) {
                        Icon(Icons.Default.Refresh, null, tint = TextSecondary)
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
                        Text("🛍️", fontSize = 52.sp)
                        Spacer(Modifier.height(12.dp))
                        Text(
                            text       = "Sin pedidos",
                            style      = MaterialTheme.typography.titleMedium,
                            fontWeight = FontWeight.Bold,
                            color      = TextPrimary,
                        )
                        Text(
                            text  = if (state.statusFilter.isBlank()) "Aún no hay pedidos"
                                    else "Sin pedidos con este estado",
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
                    verticalArrangement = Arrangement.spacedBy(10.dp),
                ) {
                    items(state.orders, key = { it.id }) { order ->
                        OrderAdminCard(
                            order    = order,
                            onStatus = { newStatus -> viewModel.changeStatus(order.id, newStatus) },
                            onClick  = { onOrderDetail(order.id) },
                        )
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

// ── OrderAdminCard ────────────────────────────────────────────

@Composable
private fun OrderAdminCard(
    order:    Order,
    onStatus: (OrderStatus) -> Unit,
    onClick:  () -> Unit,
) {
    val dateFmt   = SimpleDateFormat("dd MMM yyyy", Locale("es"))
    val inputFmt  = SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSSSS'Z'", Locale.getDefault())
    val dateStr   = runCatching { dateFmt.format(inputFmt.parse(order.createdAt)!!) }
                        .getOrDefault(order.createdAt.take(10))

    Surface(
        onClick = onClick,
        shape   = MaterialTheme.shapes.large,
        color   = Surface,
        modifier = Modifier.fillMaxWidth(),
    ) {
        Column(modifier = Modifier.padding(14.dp)) {
            // Primera fila: ID + cliente + fecha
            Row(
                modifier              = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment     = Alignment.Top,
            ) {
                Column {
                    Text(
                        text       = "Pedido #${order.id}",
                        style      = MaterialTheme.typography.titleSmall,
                        fontWeight = FontWeight.Bold,
                        color      = TextPrimary,
                    )
                    Text(
                        text  = "${order.username} · $dateStr",
                        style = MaterialTheme.typography.bodySmall,
                        color = TextSecondary,
                    )
                }
                // Selector de estado inline
                StatusDropdown(
                    current  = order.status,
                    onChange = onStatus,
                )
            }

            Spacer(Modifier.height(10.dp))

            // Preview de ítems
            Row(
                horizontalArrangement = Arrangement.spacedBy(6.dp),
                modifier              = Modifier.fillMaxWidth(),
            ) {
                order.items.take(2).forEach { item ->
                    Surface(
                        color    = Surface2,
                        shape    = MaterialTheme.shapes.extraSmall,
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
                if (order.items.size > 2) {
                    Text(
                        text  = "+${order.items.size - 2} más",
                        style = MaterialTheme.typography.bodySmall,
                        color = TextFaint,
                        modifier = Modifier.align(Alignment.CenterVertically),
                    )
                }
            }

            Spacer(Modifier.height(10.dp))
            HorizontalDivider(color = BorderLight, thickness = 0.5.dp)
            Spacer(Modifier.height(8.dp))

            // Footer: ítems + total + ver detalle
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
                Row(verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                    Text(
                        text       = "$${"%.2f".format(order.total)}",
                        style      = MaterialTheme.typography.titleSmall,
                        fontWeight = FontWeight.ExtraBold,
                        color      = Accent,
                    )
                    Icon(
                        Icons.Default.ChevronRight,
                        contentDescription = "Ver detalle",
                        tint     = TextFaint,
                        modifier = Modifier.size(18.dp),
                    )
                }
            }
        }
    }
}
```

---

## 10.4 Detalle de pedido admin — `presentation/ui/admin/orders/OrderAdminDetailScreen.kt`

```kotlin
// presentation/ui/admin/orders/OrderAdminDetailScreen.kt
package com.shopapp.presentation.ui.admin.orders

import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
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
import com.shopapp.domain.model.Order
import com.shopapp.domain.model.OrderItem
import com.shopapp.domain.model.OrderStatus
import com.shopapp.presentation.components.LoadingScreen
import com.shopapp.presentation.components.ErrorScreen
import com.shopapp.presentation.viewmodel.OrderDetailUiState
import com.shopapp.presentation.viewmodel.OrderDetailViewModel
import com.shopapp.theme.*
import java.text.SimpleDateFormat
import java.util.Locale

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun OrderAdminDetailScreen(
    orderId:    Int,
    onBack:     () -> Unit,
    onStatusChange: (Int, OrderStatus) -> Unit,
    viewModel:  OrderDetailViewModel = hiltViewModel(),
) {
    val state by viewModel.state.collectAsState()
    LaunchedEffect(orderId) { viewModel.load(orderId) }

    when (val s = state) {
        is OrderDetailUiState.Loading ->
            LoadingScreen("Cargando pedido #$orderId...")
        is OrderDetailUiState.Error   ->
            ErrorScreen(s.message, onRetry = { viewModel.load(orderId) })
        is OrderDetailUiState.Success ->
            AdminOrderDetailContent(
                order         = s.order,
                onBack        = onBack,
                onStatusChange = { newStatus ->
                    onStatusChange(orderId, newStatus)
                    viewModel.load(orderId)
                },
            )
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
private fun AdminOrderDetailContent(
    order:         Order,
    onBack:        () -> Unit,
    onStatusChange:(OrderStatus) -> Unit,
) {
    val inputFmt  = SimpleDateFormat("yyyy-MM-dd'T'HH:mm:ss.SSSSSS'Z'", Locale.getDefault())
    val outputFmt = SimpleDateFormat("dd MMM yyyy · HH:mm", Locale("es"))
    val dateStr   = runCatching { outputFmt.format(inputFmt.parse(order.createdAt)!!) }
                        .getOrDefault(order.createdAt.take(16))
    val updatedStr = runCatching { outputFmt.format(inputFmt.parse(order.updatedAt)!!) }
                        .getOrDefault(order.updatedAt.take(16))

    val taxAmount = order.total - order.total / 1.15
    val subtotal  = order.total - taxAmount

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
                        Text(
                            order.username,
                            style = MaterialTheme.typography.bodySmall,
                            color = TextSecondary,
                        )
                    }
                },
                navigationIcon = {
                    IconButton(onClick = onBack) {
                        Icon(Icons.Default.ArrowBack, null, tint = TextPrimary)
                    }
                },
                actions = {
                    // Selector de estado en la TopAppBar
                    StatusDropdown(
                        current  = order.status,
                        onChange = onStatusChange,
                        modifier = Modifier.padding(end = 12.dp),
                    )
                },
                colors = TopAppBarDefaults.topAppBarColors(containerColor = Surface),
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
            verticalArrangement = Arrangement.spacedBy(14.dp),
        ) {

            // ── Info general ──────────────────────────────────
            Surface(color = Surface, shape = MaterialTheme.shapes.large, modifier = Modifier.fillMaxWidth()) {
                Column(modifier = Modifier.padding(16.dp)) {
                    SectionLabel("Información del pedido")
                    Spacer(Modifier.height(12.dp))
                    InfoGrid(listOf(
                        "Cliente"      to order.username,
                        "Fecha"        to dateStr,
                        "Actualizado"  to updatedStr,
                        "Total ítems"  to "${order.numItems}",
                    ))
                }
            }

            // ── Ítems ──────────────────────────────────────────
            Surface(color = Surface, shape = MaterialTheme.shapes.large, modifier = Modifier.fillMaxWidth()) {
                Column(modifier = Modifier.padding(16.dp)) {
                    SectionLabel("Productos (${order.numItems})")
                    Spacer(Modifier.height(12.dp))
                    Column(verticalArrangement = Arrangement.spacedBy(10.dp)) {
                        order.items.forEach { item ->
                            AdminOrderItemRow(item = item)
                        }
                    }
                }
            }

            // ── Totales ────────────────────────────────────────
            Surface(color = Surface, shape = MaterialTheme.shapes.large, modifier = Modifier.fillMaxWidth()) {
                Column(modifier = Modifier.padding(16.dp)) {
                    SectionLabel("Resumen financiero")
                    Spacer(Modifier.height(12.dp))
                    Column(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                        FinancialRow("Subtotal (sin IVA)", subtotal, false)
                        FinancialRow("IVA (15%)",          taxAmount, false)
                        HorizontalDivider(color = Border, thickness = 0.5.dp)
                        FinancialRow("Total",              order.total, true)
                    }
                }
            }

            // ── Cambio rápido de estado ────────────────────────
            Surface(color = Surface, shape = MaterialTheme.shapes.large, modifier = Modifier.fillMaxWidth()) {
                Column(modifier = Modifier.padding(16.dp)) {
                    SectionLabel("Cambiar estado")
                    Spacer(Modifier.height(12.dp))
                    Row(
                        modifier              = Modifier.fillMaxWidth(),
                        horizontalArrangement = Arrangement.spacedBy(8.dp),
                    ) {
                        OrderStatus.entries
                            .filter { it != order.status }
                            .forEach { status ->
                                Surface(
                                    onClick   = { onStatusChange(status) },
                                    shape     = MaterialTheme.shapes.small,
                                    color     = com.shopapp.presentation.components
                                        .orderStatusColor(status).copy(alpha = 0.1f),
                                    modifier  = Modifier.weight(1f),
                                ) {
                                    Text(
                                        text     = status.label,
                                        color    = com.shopapp.presentation.components
                                            .orderStatusColor(status),
                                        fontSize = 11.sp,
                                        fontWeight = FontWeight.Bold,
                                        modifier = Modifier.padding(8.dp),
                                    )
                                }
                            }
                    }
                }
            }
        }
    }
}

// ── Sub-composables ───────────────────────────────────────────

@Composable
private fun SectionLabel(text: String) {
    Text(
        text          = text,
        style         = MaterialTheme.typography.labelSmall,
        color         = TextSecondary,
        letterSpacing = 0.8.sp,
    )
}

@Composable
private fun InfoGrid(items: List<Pair<String, String>>) {
    items.chunked(2).forEach { row ->
        Row(
            modifier              = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.spacedBy(12.dp),
        ) {
            row.forEach { (label, value) ->
                Column(modifier = Modifier.weight(1f)) {
                    Text(
                        text  = label,
                        style = MaterialTheme.typography.bodySmall,
                        color = TextSecondary,
                    )
                    Text(
                        text       = value,
                        style      = MaterialTheme.typography.bodyMedium,
                        fontWeight = FontWeight.SemiBold,
                        color      = TextPrimary,
                    )
                    Spacer(Modifier.height(10.dp))
                }
            }
            // Celda vacía si el row tiene 1 elemento
            if (row.size == 1) Spacer(Modifier.weight(1f))
        }
    }
}

@Composable
private fun AdminOrderItemRow(item: OrderItem) {
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
            style      = MaterialTheme.typography.bodyMedium,
            fontWeight = FontWeight.Bold,
            color      = Accent,
        )
    }
}

@Composable
private fun FinancialRow(label: String, value: Double, isFinal: Boolean) {
    Row(
        modifier              = Modifier.fillMaxWidth(),
        horizontalArrangement = Arrangement.SpaceBetween,
    ) {
        Text(
            text       = label,
            style      = if (isFinal) MaterialTheme.typography.titleSmall
                         else MaterialTheme.typography.bodyMedium,
            fontWeight = if (isFinal) FontWeight.Bold else FontWeight.Normal,
            color      = if (isFinal) TextPrimary else TextSecondary,
        )
        Text(
            text       = "$${"%.2f".format(value)}",
            style      = if (isFinal) MaterialTheme.typography.titleSmall
                         else MaterialTheme.typography.bodyMedium,
            fontWeight = if (isFinal) FontWeight.ExtraBold else FontWeight.SemiBold,
            color      = if (isFinal) Accent else TextPrimary,
        )
    }
}
```

---

## 10.5 Actualizar NavGraph — rutas admin de pedidos

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
                    onNavClick   = { r ->
                        navController.navigate(r) { launchSingleTop = true }
                    },
                    onStoreClick = { navController.navigate(Screen.Home.route) },
                    onLogout     = {
                        authViewModel.logout()
                        navController.navigate(Screen.Login.route) {
                            popUpTo(0) { inclusive = true }
                        }
                    },
                ) { padding ->
                    Box(
                        modifier = Modifier
                            .fillMaxSize()
                            .padding(padding),
                        contentAlignment = Alignment.Center,
                    ) {
                        Text("Usuarios — próximo módulo", color = TextSecondary)
                    }
                }
            }
        }
    }
}

```

---

## ✅ Checkpoint Módulo 10

| # | Acción | Resultado esperado |
|---|--------|-------------------|
| 1 | Drawer → "Pedidos" | Lista paginada de pedidos |
| 2 | Chip "Pendientes" | Solo pedidos con estado pending |
| 3 | Tap en `StatusDropdown` | Dropdown con 5 estados y colores |
| 4 | Cambiar a "Confirmado" | Badge cambia al instante (optimista) |
| 5 | Sin conexión → cambio de estado | Revierte al estado original |
| 6 | Tap en la card | Navega al detalle del pedido |
| 7 | Detalle muestra cliente y fechas | Grid 2×2 con info general |
| 8 | Ítems con precios | Nombre, ud., precio y subtotal |
| 9 | Desglose IVA 15% | Subtotal + IVA + Total |
| 10 | Botones de cambio rápido | Todos los estados excepto el actual |
| 11 | Verificar en vista cliente | El pedido muestra el nuevo estado |

### Verificar desde Django

```bash
uv run python manage.py shell -c "
from store.models import Order
for o in Order.objects.order_by('-updated_at')[:3]:
    print(f'#{o.id} {o.status} — {o.user.username}')
"
```

---

## Resumen

| Elemento | Estado |
|---|---|
| `OrdersAdminViewModel` con cambio optimista | ✅ |
| `StatusDropdown` con `DropdownMenu` M3 | ✅ |
| `OrdersAdminScreen` con filtros y paginación | ✅ |
| `OrderAdminCard` con selector inline | ✅ |
| `OrderAdminDetailScreen` con `TopAppBar` | ✅ |
| Detalle: info general, ítems, IVA, cambio rápido | ✅ |
| Ruta `admin/orders/{id}` en NavGraph | ✅ |
| ViewModel compartido entre lista y detalle | ✅ |

**Siguiente módulo →** M11: Admin CRUD Usuarios — lista, crear, editar, toggle staff y activar/desactivar