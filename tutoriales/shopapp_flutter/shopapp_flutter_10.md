# Flutter Shop App — Módulo 10
## Admin CRUD Pedidos — Lista con filtros, cambio de estado inline y detalle

> **Objetivo:** Gestión de pedidos desde el panel admin con filtros por estado, selector de estado inline con `DropdownButton`, paginación y pantalla de detalle con desglose de IVA.
> **Checkpoint final:** filtrar pedidos por estado "pending", cambiar a "confirmed" y verificar que el cliente ve el nuevo estado en su historial.

---

## 10.1 Provider de pedidos admin — `presentation/providers/orders_admin_provider.dart`

```dart
// lib/presentation/providers/orders_admin_provider.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../data/remote/api/order_remote_datasource.dart';
import '../../domain/model/order.dart';

class OrdersAdminState {
  final List<Order> orders;
  final bool        isLoading;
  final bool        isLoadingMore;
  final String?     error;
  final int         total;
  final bool        hasMore;
  final String      statusFilter;
  final int         page;

  const OrdersAdminState({
    this.orders        = const [],
    this.isLoading     = false,
    this.isLoadingMore = false,
    this.error,
    this.total         = 0,
    this.hasMore       = false,
    this.statusFilter  = '',
    this.page          = 1,
  });

  OrdersAdminState copyWith({
    List<Order>? orders,
    bool?        isLoading,
    bool?        isLoadingMore,
    String?      error,
    int?         total,
    bool?        hasMore,
    String?      statusFilter,
    int?         page,
  }) => OrdersAdminState(
    orders:        orders        ?? this.orders,
    isLoading:     isLoading     ?? this.isLoading,
    isLoadingMore: isLoadingMore ?? this.isLoadingMore,
    error:         error,
    total:         total         ?? this.total,
    hasMore:       hasMore       ?? this.hasMore,
    statusFilter:  statusFilter  ?? this.statusFilter,
    page:          page          ?? this.page,
  );
}

class OrdersAdminNotifier extends StateNotifier<OrdersAdminState> {
  final OrderRemoteDatasource _datasource;

  OrdersAdminNotifier(this._datasource) : super(const OrdersAdminState()) {
    load();
  }

  Future<void> load({bool reset = true}) async {
    final s    = state;
    final page = reset ? 1 : s.page;

    if (reset) {
      state = s.copyWith(isLoading: true, error: null, page: 1);
    } else {
      if (s.isLoadingMore || !s.hasMore) return;
      state = s.copyWith(isLoadingMore: true);
    }

    try {
      final result = await _datasource.getOrders(
        page:   page,
        status: s.statusFilter.isEmpty ? null : s.statusFilter,
      );
      state = state.copyWith(
        orders:        reset ? result.results : [...state.orders, ...result.results],
        total:         result.count,
        hasMore:       result.next != null,
        isLoading:     false,
        isLoadingMore: false,
        page:          page + 1,
        error:         null,
      );
    } catch (e) {
      state = state.copyWith(
        isLoading:     false,
        isLoadingMore: false,
        error:         e.toString().replaceAll('Exception: ', ''),
      );
    }
  }

  void setStatusFilter(String filter) {
    state = state.copyWith(statusFilter: filter);
    load();
  }

  void loadMore() => load(reset: false);
  void refresh()  => load();

  // Cambio optimista de estado
  Future<void> changeStatus(int orderId, OrderStatus newStatus) async {
    final prevStatus = state.orders
        .firstWhere((o) => o.id == orderId, orElse: () => state.orders.first)
        .status;

    state = state.copyWith(
      orders: state.orders.map((o) =>
        o.id == orderId ? o.copyWith(status: newStatus) : o,
      ).toList(),
    );

    try {
      await _datasource.updateStatus(orderId, newStatus.value);
    } catch (_) {
      // Revertir
      state = state.copyWith(
        orders: state.orders.map((o) =>
          o.id == orderId ? o.copyWith(status: prevStatus) : o,
        ).toList(),
      );
    }
  }
}

final ordersAdminProvider =
    StateNotifierProvider<OrdersAdminNotifier, OrdersAdminState>((ref) {
  return OrdersAdminNotifier(ref.watch(orderDatasourceProvider));
});

// Provider del detalle de pedido (admin)
final orderAdminDetailProvider = FutureProvider.family<Order, int>((ref, id) {
  return ref.watch(orderDatasourceProvider).getOrder(id);
});
```

---

## 10.2 Widget StatusDropdown — `presentation/widgets/status_dropdown.dart`

```dart
// lib/presentation/widgets/status_dropdown.dart

import 'package:flutter/material.dart';
import '../../../core/utils/formatters.dart';
import '../../domain/model/order.dart';

class StatusDropdown extends StatelessWidget {
  final OrderStatus                current;
  final void Function(OrderStatus) onChange;

  const StatusDropdown({
    super.key,
    required this.current,
    required this.onChange,
  });

  @override
  Widget build(BuildContext context) {
    final color = orderStatusColor(current);

    return Container(
      padding:    const EdgeInsets.symmetric(horizontal: 10, vertical: 4),
      decoration: BoxDecoration(
        color:        color.withValues(alpha: 0.12),
        borderRadius: BorderRadius.circular(999),
        border:       Border.all(color: color.withValues(alpha: 0.4)),
      ),
      child: DropdownButton<OrderStatus>(
        value:        current,
        isDense:      true,
        underline:    const SizedBox.shrink(),
        dropdownColor:const Color(0xFF111118),
        borderRadius: BorderRadius.circular(12),
        icon:         Icon(Icons.arrow_drop_down, color: color, size: 18),
        style:        TextStyle(
          color:      color,
          fontSize:   12,
          fontWeight: FontWeight.bold,
        ),
        selectedItemBuilder: (_) => OrderStatus.values.map((s) {
          final c = orderStatusColor(s);
          return Row(
            mainAxisSize: MainAxisSize.min,
            children: [
              Container(
                width:  6, height: 6,
                margin: const EdgeInsets.only(right: 5),
                decoration: BoxDecoration(color: c, shape: BoxShape.circle),
              ),
              Text(s.label, style: TextStyle(color: c, fontWeight: FontWeight.bold, fontSize: 12)),
            ],
          );
        }).toList(),
        items: OrderStatus.values.map((s) {
          final c     = orderStatusColor(s);
          final isCur = s == current;
          return DropdownMenuItem(
            value: s,
            child: Row(
              children: [
                Container(
                  width:  8, height: 8,
                  decoration: BoxDecoration(color: c, shape: BoxShape.circle),
                ),
                const SizedBox(width: 8),
                Text(
                  s.label,
                  style: TextStyle(
                    color:      isCur ? c : Colors.white70,
                    fontWeight: isCur ? FontWeight.bold : FontWeight.normal,
                    fontSize:   13,
                  ),
                ),
                if (isCur) ...[
                  const SizedBox(width: 4),
                  Icon(Icons.check, size: 14, color: c),
                ],
              ],
            ),
          );
        }).toList(),
        onChanged: (s) {
          if (s != null && s != current) onChange(s);
        },
      ),
    );
  }
}
```

---

## 10.3 Pantalla de pedidos admin — `presentation/screens/admin/orders_admin_screen.dart`

```dart
// lib/presentation/screens/admin/orders_admin_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/formatters.dart';
import '../../domain/model/order.dart';
import '../../widgets/status_badge.dart';
import '../../providers/orders_admin_provider.dart';
import '../../widgets/status_dropdown.dart';

const _statusFilters = [
  ('',          'Todos'),
  ('pending',   'Pendientes'),
  ('confirmed', 'Confirmados'),
  ('shipped',   'Enviados'),
  ('delivered', 'Entregados'),
  ('cancelled', 'Cancelados'),
];

class OrdersAdminScreen extends ConsumerStatefulWidget {
  const OrdersAdminScreen({super.key});

  @override
  ConsumerState<OrdersAdminScreen> createState() => _OrdersAdminScreenState();
}

class _OrdersAdminScreenState extends ConsumerState<OrdersAdminScreen> {
  final _scrollCtrl = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollCtrl.addListener(() {
      if (_scrollCtrl.position.pixels >=
          _scrollCtrl.position.maxScrollExtent - 150) {
        ref.read(ordersAdminProvider.notifier).loadMore();
      }
    });
  }

  @override
  void dispose() {
    _scrollCtrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(ordersAdminProvider);
    final tt    = Theme.of(context).textTheme;

    return Column(
      children: [
        // ── Header ──────────────────────────────────────────
        Container(
          color:   AppColors.surface,
          padding: const EdgeInsets.fromLTRB(16, 16, 16, 0),
          child:   Column(
            children: [
              Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [
                      Text('Pedidos', style: tt.headlineMedium),
                      Text('${state.total} pedidos',
                          style: const TextStyle(color: AppColors.textSecondary, fontSize: 13)),
                    ],
                  ),
                  IconButton(
                    onPressed: ref.read(ordersAdminProvider.notifier).refresh,
                    icon:      const Icon(Icons.refresh_rounded, color: AppColors.textSecondary),
                  ),
                ],
              ),
              const SizedBox(height: 12),
              SizedBox(
                height: 34,
                child:  ListView(
                  scrollDirection: Axis.horizontal,
                  children: _statusFilters.map((f) => Padding(
                    padding: const EdgeInsets.only(right: 8),
                    child:   ChoiceChip(
                      label:     Text(f.$2),
                      selected:  state.statusFilter == f.$1,
                      onSelected:(_) =>
                          ref.read(ordersAdminProvider.notifier).setStatusFilter(f.$1),
                    ),
                  )).toList(),
                ),
              ),
              const SizedBox(height: 12),
            ],
          ),
        ),

        // ── Contenido ─────────────────────────────────────────
        Expanded(
          child: Builder(builder: (_) {
            if (state.isLoading && state.orders.isEmpty) {
              return const Center(
                child: CircularProgressIndicator(color: AppColors.accent),
              );
            }
            if (state.error != null && state.orders.isEmpty) {
              return Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Text(state.error!, style: const TextStyle(color: AppColors.error)),
                    const SizedBox(height: 12),
                    ElevatedButton(
                      onPressed: ref.read(ordersAdminProvider.notifier).refresh,
                      child:     const Text('Reintentar'),
                    ),
                  ],
                ),
              );
            }
            if (state.orders.isEmpty) {
              return const Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Text('🛍️', style: TextStyle(fontSize: 52)),
                    SizedBox(height: 12),
                    Text('Sin pedidos',
                        style: TextStyle(
                          color: AppColors.textPrimary, fontSize: 18, fontWeight: FontWeight.bold,
                        )),
                  ],
                ),
              );
            }

            return ListView.separated(
              controller:      _scrollCtrl,
              padding:         const EdgeInsets.all(16),
              itemCount:       state.orders.length + (state.isLoadingMore ? 1 : 0),
              separatorBuilder:(_, __) => const SizedBox(height: 10),
              itemBuilder: (_, i) {
                if (i >= state.orders.length) {
                  return const Center(
                    child: Padding(
                      padding: EdgeInsets.all(16),
                      child:   CircularProgressIndicator(
                        color: AppColors.accent, strokeWidth: 2,
                      ),
                    ),
                  );
                }
                final order = state.orders[i];
                return _OrderAdminCard(
                  order:    order,
                  onStatus: (s) => ref.read(ordersAdminProvider.notifier)
                      .changeStatus(order.id, s),
                  onDetail: () => context.push('/admin/orders/${order.id}'),
                );
              },
            );
          }),
        ),
      ],
    );
  }
}

// ── OrderAdminCard ────────────────────────────────────────────

class _OrderAdminCard extends StatelessWidget {
  final Order                      order;
  final void Function(OrderStatus) onStatus;
  final VoidCallback               onDetail;

  const _OrderAdminCard({
    required this.order,
    required this.onStatus,
    required this.onDetail,
  });

  @override
  Widget build(BuildContext context) {
    final dateStr = formatDate(order.createdAt);

    return GestureDetector(
      onTap: onDetail,
      child: Container(
        padding:    const EdgeInsets.all(14),
        decoration: BoxDecoration(
          color:        AppColors.surface,
          borderRadius: BorderRadius.circular(14),
          border:       Border.all(color: AppColors.border),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Primera fila
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      'Pedido #${order.id}',
                      style: const TextStyle(
                        color: AppColors.textPrimary, fontWeight: FontWeight.bold, fontSize: 15,
                      ),
                    ),
                    Text(
                      '${order.username} · $dateStr',
                      style: const TextStyle(color: AppColors.textSecondary, fontSize: 12),
                    ),
                  ],
                ),
                StatusDropdown(current: order.status, onChange: onStatus),
              ],
            ),
            const SizedBox(height: 10),

            // Preview ítems
            Wrap(
              spacing: 6, runSpacing: 4,
              children: [
                ...order.items.take(2).map((item) => Container(
                  padding:    const EdgeInsets.symmetric(horizontal: 8, vertical: 3),
                  decoration: BoxDecoration(
                    color:        AppColors.surface2,
                    borderRadius: BorderRadius.circular(6),
                  ),
                  child: Text(
                    '${item.quantity}× ${item.productName}',
                    style: const TextStyle(color: AppColors.textSecondary, fontSize: 11),
                    maxLines: 1, overflow: TextOverflow.ellipsis,
                  ),
                )),
                if (order.items.length > 2)
                  Text('+${order.items.length - 2} más',
                      style: const TextStyle(color: AppColors.textFaint, fontSize: 11)),
              ],
            ),
            const SizedBox(height: 10),
            const Divider(height: 1),
            const SizedBox(height: 8),

            // Footer
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(
                  '${order.numItems} producto${order.numItems != 1 ? "s" : ""}',
                  style: const TextStyle(color: AppColors.textSecondary, fontSize: 12),
                ),
                Row(
                  children: [
                    Text(
                      formatPrice(order.total),
                      style: const TextStyle(
                        color: AppColors.accent, fontWeight: FontWeight.bold, fontSize: 15,
                      ),
                    ),
                    const SizedBox(width: 4),
                    const Icon(Icons.chevron_right, color: AppColors.textFaint, size: 18),
                  ],
                ),
              ],
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 10.4 Detalle de pedido admin — `presentation/screens/admin/order_admin_detail_screen.dart`

```dart
// lib/presentation/screens/admin/order_admin_detail_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/formatters.dart';
import '../../domain/models/order.dart';
import '../../widgets/status_badge.dart';
import '../../providers/orders_admin_provider.dart';
import '../../widgets/status_dropdown.dart';

class OrderAdminDetailScreen extends ConsumerWidget {
  final int orderId;
  const OrderAdminDetailScreen({super.key, required this.orderId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final orderAsync = ref.watch(orderAdminDetailProvider(orderId));

    return orderAsync.when(
      loading: () => const Center(
        child: CircularProgressIndicator(color: AppColors.accent),
      ),
      error: (err, _) => Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(err.toString(), style: const TextStyle(color: AppColors.error)),
            const SizedBox(height: 12),
            ElevatedButton(
              onPressed: () => context.pop(),
              child:     const Text('Volver'),
            ),
          ],
        ),
      ),
      data: (order) => _DetailContent(order: order, ref: ref),
    );
  }
}

class _DetailContent extends StatelessWidget {
  final Order  order;
  final WidgetRef ref;
  const _DetailContent({required this.order, required this.ref});

  @override
  Widget build(BuildContext context) {
    final taxAmount  = order.total - order.total / 1.15;
    final subtotal   = order.total - taxAmount;
    final dateStr    = formatDateTime(order.createdAt);
    final updatedStr = formatDateTime(order.updatedAt);

    return SingleChildScrollView(
      padding: const EdgeInsets.all(16),
      child:   Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [

          // Info general + estado
          _Card(
            title: 'Información del pedido',
            child: Column(
              children: [
                _InfoRow('Cliente',     order.username),
                _InfoRow('Fecha',       dateStr),
                _InfoRow('Actualizado', updatedStr),
                _InfoRow('Ítems',       '${order.numItems}'),
                const SizedBox(height: 12),
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    const Text('Estado actual',
                        style: TextStyle(color: AppColors.textSecondary, fontSize: 13)),
                    StatusBadge(status: order.status),
                  ],
                ),
                const SizedBox(height: 12),
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  children: [
                    const Text('Cambiar estado',
                        style: TextStyle(color: AppColors.textSecondary, fontSize: 13)),
                    StatusDropdown(
                      current:  order.status,
                      onChange: (s) {
                        ref.read(ordersAdminProvider.notifier).changeStatus(order.id, s);
                        ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                          content:         Text('Estado cambiado a "${s.label}"'),
                          backgroundColor: AppColors.success,
                          duration:        const Duration(seconds: 2),
                        ));
                      },
                    ),
                  ],
                ),
              ],
            ),
          ),
          const SizedBox(height: 14),

          // Ítems
          _Card(
            title: 'Productos (${order.numItems})',
            child: Column(
              children: order.items.map((item) => Padding(
                padding: const EdgeInsets.symmetric(vertical: 8),
                child:   Row(
                  children: [
                    Container(
                      width:  44, height: 44,
                      decoration: BoxDecoration(
                        color:        AppColors.surface2,
                        borderRadius: BorderRadius.circular(8),
                      ),
                      child: const Center(child: Text('📦', style: TextStyle(fontSize: 20))),
                    ),
                    const SizedBox(width: 12),
                    Expanded(
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Text(item.productName,
                              style: const TextStyle(
                                color: AppColors.textPrimary, fontWeight: FontWeight.w600,
                              )),
                          Text('${formatPrice(item.unitPrice)} × ${item.quantity} ud.',
                              style: const TextStyle(color: AppColors.textSecondary, fontSize: 12)),
                        ],
                      ),
                    ),
                    Text(formatPrice(item.subtotal),
                        style: const TextStyle(
                          color: AppColors.accent, fontWeight: FontWeight.bold,
                        )),
                  ],
                ),
              )).toList(),
            ),
          ),
          const SizedBox(height: 14),

          // Resumen financiero
          _Card(
            title: 'Resumen financiero',
            child: Column(
              children: [
                _TotalRow('Subtotal (sin IVA)', subtotal,    false),
                const SizedBox(height: 6),
                _TotalRow('IVA (15%)',          taxAmount,   false),
                const SizedBox(height: 8),
                const Divider(),
                const SizedBox(height: 8),
                _TotalRow('Total',              order.total, true),
              ],
            ),
          ),
          const SizedBox(height: 14),

          // Cambio rápido de estado
          _Card(
            title: 'Cambio rápido de estado',
            child: Wrap(
              spacing: 8, runSpacing: 8,
              children: OrderStatus.values
                  .where((s) => s != order.status)
                  .map((s) {
                    final color = orderStatusColor(s);
                    return GestureDetector(
                      onTap: () {
                        ref.read(ordersAdminProvider.notifier).changeStatus(order.id, s);
                        ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                          content:         Text('Estado cambiado a "${s.label}"'),
                          backgroundColor: AppColors.success,
                          duration:        const Duration(seconds: 2),
                        ));
                      },
                      child: Container(
                        padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 8),
                        decoration: BoxDecoration(
                          color:        color.withValues(alpha: 0.1),
                          borderRadius: BorderRadius.circular(10),
                          border:       Border.all(color: color.withValues(alpha: 0.3)),
                        ),
                        child: Row(
                          mainAxisSize: MainAxisSize.min,
                          children: [
                            Container(
                              width:  8, height: 8,
                              decoration: BoxDecoration(color: color, shape: BoxShape.circle),
                            ),
                            const SizedBox(width: 6),
                            Text(
                              s.label,
                              style: TextStyle(
                                color: color, fontWeight: FontWeight.bold, fontSize: 13,
                              ),
                            ),
                          ],
                        ),
                      ),
                    );
                  }).toList(),
            ),
          ),
          const SizedBox(height: 24),
        ],
      ),
    );
  }
}

// ── Sub-widgets ───────────────────────────────────────────────

class _Card extends StatelessWidget {
  final String title;
  final Widget child;
  const _Card({required this.title, required this.child});

  @override
  Widget build(BuildContext context) => Container(
    width:   double.infinity,
    padding: const EdgeInsets.all(16),
    decoration: BoxDecoration(
      color:        AppColors.surface,
      borderRadius: BorderRadius.circular(16),
    ),
    child: Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: [
        Text(
          title.toUpperCase(),
          style: const TextStyle(
            color: AppColors.textSecondary, fontSize: 11,
            fontWeight: FontWeight.bold, letterSpacing: 0.8,
          ),
        ),
        const SizedBox(height: 14),
        child,
      ],
    ),
  );
}

class _InfoRow extends StatelessWidget {
  final String label;
  final String value;
  const _InfoRow(this.label, this.value);

  @override
  Widget build(BuildContext context) => Padding(
    padding: const EdgeInsets.symmetric(vertical: 5),
    child:   Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Text(label, style: const TextStyle(color: AppColors.textSecondary, fontSize: 13)),
        Text(value, style: const TextStyle(
          color: AppColors.textPrimary, fontWeight: FontWeight.w600, fontSize: 13,
        )),
      ],
    ),
  );
}

class _TotalRow extends StatelessWidget {
  final String label;
  final double value;
  final bool   isFinal;
  const _TotalRow(this.label, this.value, this.isFinal);

  @override
  Widget build(BuildContext context) => Row(
    mainAxisAlignment: MainAxisAlignment.spaceBetween,
    children: [
      Text(
        label,
        style: TextStyle(
          color:      isFinal ? AppColors.textPrimary : AppColors.textSecondary,
          fontSize:   isFinal ? 16 : 14,
          fontWeight: isFinal ? FontWeight.bold : FontWeight.normal,
        ),
      ),
      Text(
        formatPrice(value),
        style: TextStyle(
          color:      isFinal ? AppColors.accent : AppColors.textPrimary,
          fontSize:   isFinal ? 18 : 14,
          fontWeight: isFinal ? FontWeight.w800 : FontWeight.w600,
        ),
      ),
    ],
  );
}
```

---

## 10.5 Actualizar el router

```dart
// lib/presentation/navigation/app_router.dart — reemplazar placeholders de pedidos admin

import '../screens/admin/orders_admin_screen.dart';
import '../screens/admin/order_admin_detail_screen.dart';

GoRoute(
  path:    '/admin/orders',
  builder: (_, state) => AdminShell(
    title:        'Pedidos',
    currentRoute: state.matchedLocation,
    child:        const OrdersAdminScreen(),
  ),
),
GoRoute(
  path:    '/admin/orders/:id',
  builder: (_, state) => AdminShell(
    title:        'Detalle pedido #${state.pathParameters['id']}',
    currentRoute: '/admin/orders',
    child:        OrderAdminDetailScreen(
      orderId: int.parse(state.pathParameters['id']!),
    ),
  ),
),
```

---

## ✅ Checkpoint Módulo 10

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Drawer → "Pedidos" | Lista paginada de pedidos |
| 2 | Chip "Pendientes" | Solo pedidos con estado pending |
| 3 | Tap en `StatusDropdown` | Dropdown con 5 estados y colores |
| 4 | Cambiar a "Confirmado" | Badge cambia al instante (optimista) |
| 5 | Sin conexión → cambio de estado | Revierte al estado original |
| 6 | Tap en la card | Navega al detalle del pedido |
| 7 | Detalle: cliente, fechas, ítems | Info completa del pedido |
| 8 | Desglose IVA 15% | Subtotal + IVA + Total |
| 9 | Cambio rápido de estado | Chips de colores, todos excepto el actual |
| 10 | Snackbar tras cambiar estado | "Estado cambiado a Confirmado" |
| 11 | Verificar en vista cliente | El pedido muestra el nuevo estado |
| 12 | Pull-to-refresh | Lista actualizada desde el servidor |

---

## Resumen

| Elemento | Estado |
|---|---|
| `OrdersAdminNotifier` con cambio optimista | ✅ |
| `orderAdminDetailProvider` FutureProvider.family | ✅ |
| `StatusDropdown` con `DropdownButton` coloreado | ✅ |
| `OrdersAdminScreen` con filtros y paginación | ✅ |
| `_OrderAdminCard` con selector inline | ✅ |
| `OrderAdminDetailScreen` con info, ítems, IVA | ✅ |
| Cambio rápido de estado con chips de color | ✅ |
| Snackbar de confirmación tras cada cambio | ✅ |
| Rutas `admin/orders` y `admin/orders/:id` | ✅ |

**Siguiente módulo →** M11: Admin CRUD Usuarios — lista, crear, editar, toggle staff y activar/desactivar
