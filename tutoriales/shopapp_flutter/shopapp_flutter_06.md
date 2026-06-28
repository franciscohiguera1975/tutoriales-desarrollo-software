# Flutter Shop App — Módulo 6
## Mis pedidos — Historial paginado, detalle con barra de progreso y perfil

> **Objetivo:** Zona privada del cliente completa: historial de pedidos con filtros por estado, detalle con barra de progreso visual y pantalla de perfil con logout.
> **Checkpoint final:** ver el pedido creado en M5 con estado "confirmed", barra de progreso con el nodo iluminado y perfil del usuario logueado.

---

## 6.1 Provider de pedidos — `presentation/providers/orders_provider.dart`

```dart
// lib/presentation/providers/orders_provider.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../data/remote/api/order_remote_datasource.dart';
import '../../domain/model/order.dart';

// ── Estado del historial ──────────────────────────────────────
class OrdersState {
  final List<Order> orders;
  final bool isLoading;
  final bool isLoadingMore;
  final String? error;
  final int total;
  final bool hasMore;
  final String statusFilter;
  final int page;

  const OrdersState({
    this.orders = const [],
    this.isLoading = false,
    this.isLoadingMore = false,
    this.error,
    this.total = 0,
    this.hasMore = false,
    this.statusFilter = '',
    this.page = 1,
  });

  OrdersState copyWith({
    List<Order>? orders,
    bool? isLoading,
    bool? isLoadingMore,
    String? error,
    int? total,
    bool? hasMore,
    String? statusFilter,
    int? page,
  }) => OrdersState(
    orders: orders ?? this.orders,
    isLoading: isLoading ?? this.isLoading,
    isLoadingMore: isLoadingMore ?? this.isLoadingMore,
    error: error,
    total: total ?? this.total,
    hasMore: hasMore ?? this.hasMore,
    statusFilter: statusFilter ?? this.statusFilter,
    page: page ?? this.page,
  );
}

class OrdersNotifier extends StateNotifier<OrdersState> {
  final OrderRemoteDatasource _datasource;

  OrdersNotifier(this._datasource) : super(const OrdersState()) {
    load();
  }

  Future<void> load({bool reset = true}) async {
    final s = state;
    final page = reset ? 1 : s.page;

    if (reset) {
      state = s.copyWith(isLoading: true, error: null, page: 1);
    } else {
      if (s.isLoadingMore || !s.hasMore) return;
      state = s.copyWith(isLoadingMore: true);
    }

    try {
      final result = await _datasource.getOrders(
        page: page,
        status: s.statusFilter.isEmpty ? null : s.statusFilter,
      );
      state = state.copyWith(
        orders: reset ? result.results : [...state.orders, ...result.results],
        total: result.count,
        hasMore: result.next != null,
        isLoading: false,
        isLoadingMore: false,
        page: page + 1,
        error: null,
      );
    } catch (e) {
      state = state.copyWith(
        isLoading: false,
        isLoadingMore: false,
        error: e.toString().replaceAll('Exception: ', ''),
      );
    }
  }

  void setStatusFilter(String filter) {
    state = state.copyWith(statusFilter: filter);
    load();
  }

  void loadMore() => load(reset: false);
  void refresh() => load();
}

final ordersProvider =
    StateNotifierProvider<OrdersNotifier, OrdersState>((ref) {
  return OrdersNotifier(ref.watch(orderDatasourceProvider));
});

// Provider de un pedido individual
final orderDetailProvider = FutureProvider.family<Order, int>((ref, id) {
  return ref.watch(orderDatasourceProvider).getOrder(id);
});
```

---

## 6.2 Widget StatusBadge — `presentation/widgets/status_badge.dart`

```dart
// lib/presentation/widgets/status_badge.dart

import 'package:flutter/material.dart';
import '../../core/utils/formatters.dart';
import '../../domain/model/order.dart';

class StatusBadge extends StatelessWidget {
  final OrderStatus status;
  final bool        small;

  const StatusBadge({super.key, required this.status, this.small = false});

  @override
  Widget build(BuildContext context) {
    final color = orderStatusColor(status);
    final label = status.label;

    return Container(
      padding: EdgeInsets.symmetric(
        horizontal: small ? 8  : 10,
        vertical:   small ? 3  : 5,
      ),
      decoration: BoxDecoration(
        color:        color.withValues(alpha: 0.12),
        borderRadius: BorderRadius.circular(999),
        border:       Border.all(color: color.withValues(alpha: 0.4)),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          Container(
            width:  small ? 5 : 6,
            height: small ? 5 : 6,
            decoration: BoxDecoration(color: color, shape: BoxShape.circle),
          ),
          const SizedBox(width: 5),
          Text(
            label,
            style: TextStyle(
              color:         color,
              fontSize:      small ? 10 : 11,
              fontWeight:    FontWeight.bold,
              letterSpacing: 0.3,
            ),
          ),
        ],
      ),
    );
  }
}
```

---

## 6.3 Pantalla historial — `presentation/screens/orders/orders_screen.dart`

```dart
// lib/presentation/screens/orders/orders_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/formatters.dart';
import '../../../domain/model/order.dart';
import '../../providers/orders_provider.dart';
import '../../widgets/status_badge.dart';

const _statusFilters = [
  ('',          'Todos'),
  ('pending',   'Pendientes'),
  ('confirmed', 'Confirmados'),
  ('shipped',   'Enviados'),
  ('delivered', 'Entregados'),
  ('cancelled', 'Cancelados'),
];

class OrdersScreen extends ConsumerStatefulWidget {
  const OrdersScreen({super.key});

  @override
  ConsumerState<OrdersScreen> createState() => _OrdersScreenState();
}

class _OrdersScreenState extends ConsumerState<OrdersScreen> {
  final _scrollCtrl = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollCtrl.addListener(() {
      if (_scrollCtrl.position.pixels >=
          _scrollCtrl.position.maxScrollExtent - 150) {
        ref.read(ordersProvider.notifier).loadMore();
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
    final state = ref.watch(ordersProvider);
    final tt    = Theme.of(context).textTheme;

    return Scaffold(
      body: SafeArea(
        child: Column(
          children: [
            // ── Header ──────────────────────────────────────
            Container(
              color:   AppColors.surface,
              padding: const EdgeInsets.fromLTRB(20, 20, 20, 0),
              child:   Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceBetween,
                    children: [
                      Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: [
                          Text('Mis pedidos', style: tt.headlineMedium),
                          Text(
                            '${state.total} pedido${state.total != 1 ? "s" : ""}',
                            style: tt.bodySmall,
                          ),
                        ],
                      ),
                      IconButton(
                        onPressed: ref.read(ordersProvider.notifier).refresh,
                        icon: const Icon(Icons.refresh_rounded, color: AppColors.textSecondary),
                      ),
                    ],
                  ),
                  const SizedBox(height: 12),

                  // Filtros por estado
                  SizedBox(
                    height: 34,
                    child:  ListView(
                      scrollDirection:  Axis.horizontal,
                      children: _statusFilters.map((filter) {
                        final isSelected = state.statusFilter == filter.$1;
                        return Padding(
                          padding: const EdgeInsets.only(right: 8),
                          child:   ChoiceChip(
                            label:     Text(filter.$2),
                            selected:  isSelected,
                            onSelected:(_) =>
                                ref.read(ordersProvider.notifier).setStatusFilter(filter.$1),
                          ),
                        );
                      }).toList(),
                    ),
                  ),
                  const SizedBox(height: 12),
                ],
              ),
            ),

            // ── Contenido ─────────────────────────────────────
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
                        const Text('❌', style: TextStyle(fontSize: 40)),
                        const SizedBox(height: 12),
                        Text(state.error!,
                            style: const TextStyle(color: AppColors.error)),
                        const SizedBox(height: 16),
                        ElevatedButton(
                          onPressed: ref.read(ordersProvider.notifier).refresh,
                          child: const Text('Reintentar'),
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
                        Text('📦', style: TextStyle(fontSize: 52)),
                        SizedBox(height: 16),
                        Text('Sin pedidos',
                            style: TextStyle(
                              color: AppColors.textPrimary,
                              fontSize: 18,
                              fontWeight: FontWeight.bold,
                            )),
                        Text('Tus pedidos aparecerán aquí',
                            style: TextStyle(color: AppColors.textSecondary)),
                      ],
                    ),
                  );
                }

                return ListView.separated(
                  controller:      _scrollCtrl,
                  padding:         const EdgeInsets.all(16),
                  itemCount:       state.orders.length + (state.isLoadingMore ? 1 : 0),
                  separatorBuilder:(_, __) => const SizedBox(height: 12),
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
                    return _OrderCard(
                      order:   order,
                      onTap:   () => context.push('/orders/${order.id}'),
                    );
                  },
                );
              }),
            ),
          ],
        ),
      ),
    );
  }
}

// ── OrderCard ─────────────────────────────────────────────────

class _OrderCard extends StatelessWidget {
  final Order        order;
  final VoidCallback onTap;
  const _OrderCard({required this.order, required this.onTap});

  @override
  Widget build(BuildContext context) {
    final tt      = Theme.of(context).textTheme;
    final dateStr = formatDate(order.createdAt);

    return GestureDetector(
      onTap: onTap,
      child: Container(
        padding:    const EdgeInsets.all(16),
        decoration: BoxDecoration(
          color:        AppColors.surface,
          borderRadius: BorderRadius.circular(16),
          border:       Border.all(color: AppColors.border),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // Header
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text('Pedido #${order.id}', style: tt.titleMedium),
                    Text(dateStr, style: tt.bodySmall),
                  ],
                ),
                StatusBadge(status: order.status),
              ],
            ),
            const SizedBox(height: 12),

            // Preview de ítems
            Wrap(
              spacing: 6, runSpacing: 4,
              children: [
                ...order.items.take(3).map((item) => Container(
                  padding:    const EdgeInsets.symmetric(horizontal: 8, vertical: 3),
                  decoration: BoxDecoration(
                    color:        AppColors.surface2,
                    borderRadius: BorderRadius.circular(6),
                    border:       Border.all(color: AppColors.borderLight),
                  ),
                  child: Text(
                    '${item.quantity}× ${item.productName}',
                    style: const TextStyle(color: AppColors.textSecondary, fontSize: 11),
                    maxLines: 1, overflow: TextOverflow.ellipsis,
                  ),
                )),
                if (order.items.length > 3)
                  Text(
                    '+${order.items.length - 3} más',
                    style: const TextStyle(color: AppColors.textFaint, fontSize: 11),
                  ),
              ],
            ),
            const SizedBox(height: 12),
            const Divider(height: 1),
            const SizedBox(height: 10),

            // Footer
            Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Text(
                  '${order.numItems} producto${order.numItems != 1 ? "s" : ""}',
                  style: tt.bodySmall,
                ),
                Row(
                  children: [
                    Text(
                      formatPrice(order.total),
                      style: const TextStyle(
                        color: AppColors.accent, fontSize: 16, fontWeight: FontWeight.bold,
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

## 6.4 Pantalla detalle de pedido — `presentation/screens/orders/order_detail_screen.dart`

```dart
// lib/presentation/screens/orders/order_detail_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/formatters.dart';
import '../../../domain/model/order.dart';
import '../../providers/orders_provider.dart';
import '../../widgets/status_badge.dart';

const _progressSteps = [
  OrderStatus.pending,
  OrderStatus.confirmed,
  OrderStatus.shipped,
  OrderStatus.delivered,
];

class OrderDetailScreen extends ConsumerWidget {
  final int orderId;
  const OrderDetailScreen({super.key, required this.orderId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final orderAsync = ref.watch(orderDetailProvider(orderId));

    return Scaffold(
      appBar: AppBar(
        title:   Text('Pedido #$orderId'),
        leading: IconButton(
          icon:      const Icon(Icons.arrow_back),
          onPressed: () => context.pop(),
        ),
      ),
      body: orderAsync.when(
        loading: () => const Center(
          child: CircularProgressIndicator(color: AppColors.accent),
        ),
        error: (err, _) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Text('❌', style: TextStyle(fontSize: 40)),
              const SizedBox(height: 12),
              Text(err.toString(), style: const TextStyle(color: AppColors.error)),
              const SizedBox(height: 16),
              ElevatedButton(
                onPressed: () => context.pop(),
                child:     const Text('Volver'),
              ),
            ],
          ),
        ),
        data: (order) {
          final isCancelled  = order.status == OrderStatus.cancelled;
          final currentStep  = _progressSteps.indexOf(order.status);
          final taxAmount    = order.total - order.total / 1.15;
          final subtotal     = order.total - taxAmount;
          final dateStr      = formatDateTime(order.createdAt);
          final updatedStr   = formatDateTime(order.updatedAt);

          return SingleChildScrollView(
            padding: const EdgeInsets.all(16),
            child:   Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [

                // Header con status
                Row(
                  mainAxisAlignment: MainAxisAlignment.spaceBetween,
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Text(dateStr,
                            style: const TextStyle(color: AppColors.textSecondary, fontSize: 13)),
                        const SizedBox(height: 4),
                        Text('Cliente: ${order.username}',
                            style: const TextStyle(color: AppColors.textSecondary, fontSize: 13)),
                      ],
                    ),
                    StatusBadge(status: order.status),
                  ],
                ),
                const SizedBox(height: 20),

                // ── Barra de progreso ────────────────────────
                if (!isCancelled) ...[
                  _SectionCard(
                    title: 'Estado del pedido',
                    child: _OrderProgressBar(
                      steps:       _progressSteps,
                      currentStep: currentStep,
                    ),
                  ),
                  const SizedBox(height: 14),
                ] else ...[
                  Container(
                    width:   double.infinity,
                    padding: const EdgeInsets.all(14),
                    decoration: BoxDecoration(
                      color:        AppColors.error.withValues(alpha: 0.08),
                      borderRadius: BorderRadius.circular(12),
                      border:       Border.all(color: AppColors.error.withValues(alpha: 0.3)),
                    ),
                    child: const Text(
                      '⚠️ Este pedido fue cancelado',
                      style: TextStyle(color: AppColors.error, fontWeight: FontWeight.w600),
                    ),
                  ),
                  const SizedBox(height: 14),
                ],

                // ── Productos ────────────────────────────────
                _SectionCard(
                  title: 'Productos (${order.numItems})',
                  child: Column(
                    children: order.items.map((item) => Padding(
                      padding: const EdgeInsets.symmetric(vertical: 8),
                      child: Row(
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
                                Text(
                                  '${formatPrice(item.unitPrice)} × ${item.quantity} ud.',
                                  style: const TextStyle(
                                    color: AppColors.textSecondary, fontSize: 12,
                                  ),
                                ),
                              ],
                            ),
                          ),
                          Text(
                            formatPrice(item.subtotal),
                            style: const TextStyle(
                              color: AppColors.accent, fontWeight: FontWeight.bold,
                            ),
                          ),
                        ],
                      ),
                    )).toList(),
                  ),
                ),
                const SizedBox(height: 14),

                // ── Resumen financiero ───────────────────────
                _SectionCard(
                  title: 'Resumen',
                  child: Column(
                    children: [
                      _FinancialRow('Subtotal (sin IVA)', subtotal,      false),
                      const SizedBox(height: 6),
                      _FinancialRow('IVA (15%)',          taxAmount,     false),
                      const SizedBox(height: 8),
                      const Divider(),
                      const SizedBox(height: 8),
                      _FinancialRow('Total',              order.total,   true),
                    ],
                  ),
                ),
                const SizedBox(height: 14),

                // Meta
                Center(
                  child: Text(
                    'Actualizado: $updatedStr',
                    style: const TextStyle(color: AppColors.textFaint, fontSize: 11),
                  ),
                ),
                const SizedBox(height: 24),
              ],
            ),
          );
        },
      ),
    );
  }
}

// ── Barra de progreso ─────────────────────────────────────────

class _OrderProgressBar extends StatelessWidget {
  final List<OrderStatus> steps;
  final int               currentStep;
  const _OrderProgressBar({required this.steps, required this.currentStep});

  @override
  Widget build(BuildContext context) {
    return Row(
      children: steps.asMap().entries.map((entry) {
        final idx     = entry.key;
        final step    = entry.value;
        final isDone  = idx <= currentStep;
        final isCurr  = idx == currentStep;

        return Expanded(
          child: Row(
            children: [
              // Nodo
              Column(
                children: [
                  AnimatedContainer(
                    duration: const Duration(milliseconds: 300),
                    width:  isCurr ? 36 : 30,
                    height: isCurr ? 36 : 30,
                    decoration: BoxDecoration(
                      color:  isDone ? AppColors.accent : AppColors.surface2,
                      shape:  BoxShape.circle,
                      border: Border.all(
                        color: isDone ? AppColors.accent : AppColors.border,
                        width: isCurr ? 2 : 1,
                      ),
                      boxShadow: isCurr
                          ? [BoxShadow(
                              color:       AppColors.accent.withValues(alpha: 0.3),
                              blurRadius:  8,
                              spreadRadius:2,
                            )]
                          : null,
                    ),
                    child: Center(
                      child: isDone
                          ? Text(
                              '✓',
                              style: TextStyle(
                                color:      AppColors.onAccent,
                                fontWeight: FontWeight.bold,
                                fontSize:   isCurr ? 14 : 12,
                              ),
                            )
                          : Text(
                              '${idx + 1}',
                              style: const TextStyle(
                                color:    AppColors.textFaint,
                                fontSize: 11,
                              ),
                            ),
                    ),
                  ),
                  const SizedBox(height: 6),
                  SizedBox(
                    width: 60,
                    child: Text(
                      step.label,
                      style: TextStyle(
                        color:      isDone ? AppColors.accent : AppColors.textFaint,
                        fontSize:   9,
                        fontWeight: isCurr ? FontWeight.bold : FontWeight.normal,
                      ),
                      textAlign: TextAlign.center,
                    ),
                  ),
                ],
              ),
              // Línea conectora
              if (idx < steps.length - 1)
                Expanded(
                  child: Container(
                    height: 2,
                    margin: const EdgeInsets.only(bottom: 20),
                    color:  idx < currentStep ? AppColors.accent : AppColors.border,
                  ),
                ),
            ],
          ),
        );
      }).toList(),
    );
  }
}

// ── Sub-widgets ───────────────────────────────────────────────

class _SectionCard extends StatelessWidget {
  final String title;
  final Widget child;
  const _SectionCard({required this.title, required this.child});

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

class _FinancialRow extends StatelessWidget {
  final String label;
  final double value;
  final bool   isFinal;
  const _FinancialRow(this.label, this.value, this.isFinal);

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

## 6.5 Pantalla de perfil — `presentation/screens/auth/profile_screen.dart`

```dart
// lib/presentation/screens/auth/profile_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../providers/auth_provider.dart';

class ProfileScreen extends ConsumerWidget {
  const ProfileScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final user = ref.watch(authProvider).user;
    final tt   = Theme.of(context).textTheme;

    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(24),
          child: Column(
            children: [
              const SizedBox(height: 24),

              // Avatar
              Container(
                width:  80, height: 80,
                decoration: BoxDecoration(
                  gradient: const LinearGradient(
                    colors: [AppColors.accent, AppColors.accentLight],
                    begin:  Alignment.topLeft,
                    end:    Alignment.bottomRight,
                  ),
                  shape: BoxShape.circle,
                ),
                child: Center(
                  child: Text(
                    (user?.username.isNotEmpty == true)
                        ? user!.username[0].toUpperCase()
                        : '?',
                    style: const TextStyle(
                      color:      AppColors.onAccent,
                      fontSize:   34,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                ),
              ),
              const SizedBox(height: 16),
              Text(user?.username ?? '—', style: tt.headlineMedium),
              Text(user?.email    ?? '—', style: tt.bodyMedium),
              const SizedBox(height: 8),
              if (user?.isStaff == true)
                Container(
                  padding:    const EdgeInsets.symmetric(horizontal: 14, vertical: 4),
                  decoration: BoxDecoration(
                    color:        AppColors.accent.withValues(alpha: 0.15),
                    borderRadius: BorderRadius.circular(999),
                  ),
                  child: const Text(
                    'Staff',
                    style: TextStyle(
                      color:         AppColors.accent,
                      fontSize:      12,
                      fontWeight:    FontWeight.bold,
                      letterSpacing: 0.8,
                    ),
                  ),
                ),
              const SizedBox(height: 32),

              // Info
              Container(
                width:   double.infinity,
                padding: const EdgeInsets.all(20),
                decoration: BoxDecoration(
                  color:        AppColors.surface,
                  borderRadius: BorderRadius.circular(16),
                ),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    const Text(
                      'INFORMACIÓN DE LA CUENTA',
                      style: TextStyle(
                        color:         AppColors.textSecondary,
                        fontSize:      11,
                        fontWeight:    FontWeight.bold,
                        letterSpacing: 0.8,
                      ),
                    ),
                    const SizedBox(height: 16),
                    ...[
                      ('ID de usuario', user?.id.toString() ?? '—'),
                      ('Usuario',       user?.username      ?? '—'),
                      ('Email',         user?.email         ?? '—'),
                      ('Rol',           user?.isStaff == true ? 'Administrador' : 'Cliente'),
                    ].asMap().entries.map((entry) {
                      final isLast = entry.key == 3;
                      return Column(
                        children: [
                          Padding(
                            padding: const EdgeInsets.symmetric(vertical: 10),
                            child: Row(
                              mainAxisAlignment: MainAxisAlignment.spaceBetween,
                              children: [
                                Text(entry.value.$1,
                                    style: const TextStyle(color: AppColors.textSecondary)),
                                Text(
                                  entry.value.$2,
                                  style: const TextStyle(
                                    color: AppColors.textPrimary, fontWeight: FontWeight.w600,
                                  ),
                                ),
                              ],
                            ),
                          ),
                          if (!isLast) const Divider(height: 1),
                        ],
                      );
                    }),
                  ],
                ),
              ),
              const SizedBox(height: 24),

              // Botón logout
              _LogoutButton(
                onConfirm: () async {
                  await ref.read(authProvider.notifier).logout();
                  if (context.mounted) context.go('/login');
                },
              ),
              const SizedBox(height: 32),
            ],
          ),
        ),
      ),
    );
  }
}

class _LogoutButton extends StatelessWidget {
  final Future<void> Function() onConfirm;
  const _LogoutButton({required this.onConfirm});

  @override
  Widget build(BuildContext context) => SizedBox(
    width:  double.infinity,
    height: 52,
    child:  OutlinedButton.icon(
      onPressed: () => showDialog(
        context: context,
        builder: (_) => AlertDialog(
          backgroundColor: AppColors.surface,
          shape:           RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
          title:           const Text('¿Cerrar sesión?',
              style: TextStyle(color: AppColors.textPrimary)),
          content:         const Text(
            'Tu sesión se cerrará en este dispositivo.',
            style: TextStyle(color: AppColors.textSecondary),
          ),
          actions: [
            TextButton(
 
              onPressed: () => Navigator.pop(context),
              child:     const Text('Cancelar'),
            ),
            TextButton(
              onPressed: () async {
                Navigator.pop(context);
                await onConfirm();
              },
              child: const Text(
                'Cerrar sesión',
                style: TextStyle(color: AppColors.error, fontWeight: FontWeight.bold),
              ),
            ),
          ],
        ),
      ),
      icon:  const Icon(Icons.logout, color: AppColors.error),
      label: const Text('Cerrar sesión'),
      style: OutlinedButton.styleFrom(
        foregroundColor: AppColors.error,
        side:            BorderSide(color: AppColors.error.withValues(alpha: 0.5)),
      ),
    ),
  );
}
```

---

## 6.6 Actualizar el router — reemplazar placeholders

```dart
// lib/presentation/navigation/app_router.dart — dentro del ShellRoute, reemplazar:

import '../screens/orders/orders_screen.dart';
import '../screens/orders/order_detail_screen.dart';
import '../screens/auth/profile_screen.dart';

// En routes del ShellRoute:
GoRoute(
  path:    '/orders',
  builder: (_, __) => const OrdersScreen(),
),
GoRoute(
  path:    '/orders/:id',
  builder: (_, s) => OrderDetailScreen(
    orderId: int.parse(s.pathParameters['id']!),
  ),
),
GoRoute(
  path:    '/profile',
  builder: (_, __) => const ProfileScreen(),
),
```

---

## Carrito — `presentation/providers/cart_provider.dart` y `presentation/screens/cart/cart_screen.dart`

> **Objetivo:** Mostrar el contenido real del carrito usando el `cartProvider` del proyecto y proporcionar una pantalla `CartScreen` que consuma ese provider.

```dart
// lib/presentation/screens/cart/cart_screen.dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../providers/cart_provider.dart';
import '../../../data/remote/api/order_remote_datasource.dart';

class CartScreen extends ConsumerWidget {
  const CartScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(cartProvider);

    if (state.items.isEmpty) {
      return Scaffold(
        appBar: AppBar(
          title: const Text('Carrito'),
          actions: [
            IconButton(
              icon: const Icon(Icons.receipt_long),
              tooltip: 'Mis órdenes',
              onPressed: () => context.go('/orders'),
            ),
          ],
        ),
        body: const Center(child: Text('El carrito está vacío', style: TextStyle(fontSize: 18))),
      );
    }

    return Scaffold(
      appBar: AppBar(
        title: const Text('Carrito'),
        actions: [
          IconButton(
            icon: const Icon(Icons.receipt_long),
            tooltip: 'Mis órdenes',
            onPressed: () => context.go('/orders'),
          ),
        ],
      ),
      body: ListView.separated(
        padding: const EdgeInsets.all(16),
        itemCount: state.items.length,
        separatorBuilder: (_, __) => const SizedBox(height: 12),
        itemBuilder: (_, i) {
          final it = state.items[i];
          return ListTile(
            leading: const Icon(Icons.shopping_bag),
            title: Text(it.product.name),
            subtitle: Text('${it.quantity} × ${it.product.price.toStringAsFixed(2)}'),
            trailing: Text('\$${it.subtotal.toStringAsFixed(2)}', style: const TextStyle(fontWeight: FontWeight.bold)),
          );
        },
      ),
      bottomNavigationBar: Container(
        padding: const EdgeInsets.all(16),
        color: AppColors.surface,
        child: Row(
          mainAxisAlignment: MainAxisAlignment.spaceBetween,
          children: [
            Text('Total: \$${state.totalWithTax.toStringAsFixed(2)}', style: const TextStyle(fontSize: 16, fontWeight: FontWeight.bold)),
            ElevatedButton(
              onPressed: () async {
                await _confirmOrder(context, ref, state);
              },
              style: ElevatedButton.styleFrom(minimumSize: const Size(160, 52)),
              child: const Text('Confirmar orden'),
            ),
          ],
        ),
      ),
    );
  }
}

Future<void> _confirmOrder(BuildContext context, WidgetRef ref, CartState state) async {
  final shouldConfirm = await showDialog<bool>(
    context: context,
    builder: (_) => AlertDialog(
      title: const Text('Confirmar orden'),
      content: const Text('¿Deseas confirmar esta orden y enviar los productos?'),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context, false),
          child: const Text('Cancelar'),
        ),
        ElevatedButton(
          onPressed: () => Navigator.pop(context, true),
          style: ElevatedButton.styleFrom(minimumSize: const Size(120, 48)),
          child: const Text('Confirmar'),
        ),
      ],
    ),
  );

  if (shouldConfirm != true) return;

  final datasource = ref.read(orderDatasourceProvider);

  try {
    final order = await datasource.createOrder();
    for (final item in state.items) {
      await datasource.addItem(order.id, item.product.id, item.quantity);
    }
    final confirmedOrder = await datasource.confirmOrder(order.id);

    ref.read(cartProvider.notifier).clearCart();
    if (!context.mounted) return;
    final messenger = ScaffoldMessenger.of(context);
    messenger.showSnackBar(
      SnackBar(content: Text('Orden #${confirmedOrder.id} confirmada')),
    );
    context.go('/orders');
  } catch (e) {
    if (!context.mounted) return;
    final messenger = ScaffoldMessenger.of(context);
    messenger.showSnackBar(
      SnackBar(content: Text('Error confirmando orden: $e')),
    );
  }
}
```

En producción, el botón `Confirmar orden` debería disparar el flujo real de confirmación: crear la orden en el backend, agregar los ítems del carrito, confirmar la orden y limpiar el carrito con `ref.read(cartProvider.notifier).clearCart()`.

### Integración rápida

- Añade la ruta en `lib/presentation/navigation/app_router.dart`:

```dart
import '../screens/cart/cart_screen.dart';

GoRoute(
  path: '/cart',
  builder: (_, __) => const CartScreen(),
),
```

- Llama al notifier desde `product_detail_screen.dart` al añadir producto:

```dart
ref.read(cartProvider.notifier).addItem(product, quantity: 1);
```

La `IconButton` del `AppBar` en `CartScreen` abre `'/orders'` — así agregamos el botón "Mis órdenes" solicitado.


## ✅ Checkpoint Módulo 6

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Tab "Pedidos" | Lista de pedidos del usuario |
| 2 | Pedido del M5 visible | Card con estado "Confirmado" en azul |
| 3 | Chip "Confirmados" | Solo pedidos confirmados |
| 4 | Tap en la card | Navega al detalle |
| 5 | Barra de progreso | Nodo "Confirmado" iluminado en dorado |
| 6 | Pedido cancelado | Banner rojo, sin barra |
| 7 | Ítems del pedido | Nombre, precio unitario × cantidad, subtotal |
| 8 | Total con IVA desglosado | Subtotal + IVA 15% + Total |
| 9 | Tab "Perfil" | Avatar con inicial, username, email, rol |
| 10 | "Cerrar sesión" | `AlertDialog` de confirmación |
| 11 | Confirmar logout | Navega a Login, tokens eliminados |
| 12 | Reabrir la app tras logout | Empieza en LoginScreen |

---

## Resumen

| Elemento | Estado |
|---|---|
| `OrdersNotifier` con paginación y filtros | ✅ |
| `orderDetailProvider` FutureProvider.family | ✅ |
| `StatusBadge` con color por estado | ✅ |
| `OrdersScreen` con chips de filtro | ✅ |
| `_OrderCard` con preview de ítems | ✅ |
| `OrderDetailScreen` con `_OrderProgressBar` | ✅ |
| Barra de progreso animada (4 nodos) | ✅ |
| Desglose IVA 15% en el detalle | ✅ |
| `ProfileScreen` con avatar y diálogo de logout | ✅ |
| Rutas `/orders`, `/orders/:id`, `/profile` | ✅ |

**Siguiente módulo →** M7: Panel Admin — NavigationDrawer + Dashboard con KPIs en paralelo
