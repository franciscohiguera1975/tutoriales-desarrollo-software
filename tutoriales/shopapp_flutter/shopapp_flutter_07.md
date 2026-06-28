# Flutter Shop App — Módulo 7
## Panel Admin — NavigationDrawer y Dashboard con KPIs

> **Objetivo:** Layout de administración con `NavigationDrawer` deslizable, `AppBar` contextual, Dashboard con KPIs cargados en paralelo, gráfica de barras por estado y alertas de stock bajo.
> **Checkpoint final:** login con cuenta staff → Dashboard con métricas reales del backend, gráfica de pedidos y lista de productos con stock bajo.

---

## 7.1 AdminShell — `presentation/widgets/admin_shell.dart`

```dart
// lib/presentation/widgets/admin_shell.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../theme/app_colors.dart';
import '../../providers/auth_provider.dart';

class AdminNavItem {
  final String   label;
  final IconData icon;
  final String   route;
  const AdminNavItem({required this.label, required this.icon, required this.route});
}

const adminNavItems = [
  AdminNavItem(label: 'Dashboard',  icon: Icons.dashboard_outlined,       route: '/admin'),
  AdminNavItem(label: 'Categorías', icon: Icons.category_outlined,        route: '/admin/categories'),
  AdminNavItem(label: 'Productos',  icon: Icons.inventory_2_outlined,     route: '/admin/products'),
  AdminNavItem(label: 'Pedidos',    icon: Icons.shopping_bag_outlined,    route: '/admin/orders'),
  AdminNavItem(label: 'Usuarios',   icon: Icons.people_outline,           route: '/admin/users'),
];

class AdminShell extends ConsumerWidget {
  final Widget child;
  final String title;
  final String currentRoute;

  const AdminShell({
    super.key,
    required this.child,
    required this.title,
    required this.currentRoute,
  });

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final user = ref.watch(authProvider).user;

    return Scaffold(
      appBar: AppBar(
        title: Text(title),
        actions: [
          TextButton(
            onPressed: () => context.go('/'),
            child: const Text('← Tienda',
                style: TextStyle(color: AppColors.accent, fontSize: 13)),
          ),
        ],
      ),
      drawer: NavigationDrawer(
        selectedIndex: adminNavItems.indexWhere(
          (i) => currentRoute == i.route || currentRoute.startsWith('${i.route}/'),
        ),
        onDestinationSelected: (idx) {
          Navigator.pop(context); // cerrar drawer
          context.go(adminNavItems[idx].route);
        },
        children: [
          // Header del drawer
          Container(
            color:   AppColors.surface2,
            padding: const EdgeInsets.fromLTRB(20, 48, 20, 20),
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                // Avatar
                Row(
                  children: [
                    Container(
                      width:  48, height: 48,
                      decoration: const BoxDecoration(
                        gradient: LinearGradient(
                          colors: [AppColors.accent, AppColors.accentLight],
                        ),
                        shape: BoxShape.circle,
                      ),
                      child: Center(
                        child: Text(
                          user?.username.isNotEmpty == true
                              ? user!.username[0].toUpperCase()
                              : 'A',
                          style: const TextStyle(
                            color:      AppColors.onAccent,
                            fontWeight: FontWeight.bold,
                            fontSize:   20,
                          ),
                        ),
                      ),
                    ),
                    const SizedBox(width: 12),
                    Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        Text(
                          user?.username ?? '—',
                          style: const TextStyle(
                            color:      AppColors.textPrimary,
                            fontWeight: FontWeight.bold,
                            fontSize:   16,
                          ),
                        ),
                        Container(
                          padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 2),
                          decoration: BoxDecoration(
                            color:        AppColors.accent.withValues(alpha: 0.15),
                            borderRadius: BorderRadius.circular(999),
                          ),
                          child: const Text(
                            'Staff',
                            style: TextStyle(
                              color:      AppColors.accent,
                              fontSize:   11,
                              fontWeight: FontWeight.bold,
                            ),
                          ),
                        ),
                      ],
                    ),
                  ],
                ),
                const SizedBox(height: 12),
                const Text(
                  'Panel de administración',
                  style: TextStyle(color: AppColors.textSecondary, fontSize: 12),
                ),
              ],
            ),
          ),
          const Divider(height: 1),
          const SizedBox(height: 8),

          // Items de navegación
          ...adminNavItems.map((item) => NavigationDrawerDestination(
            icon:             Icon(item.icon),
            selectedIcon:     Icon(item.icon, color: AppColors.accent),
            label:            Text(item.label),
          )),

          const Divider(),

          // Cerrar sesión
          ListTile(
            leading: const Icon(Icons.logout, color: AppColors.error),
            title:   const Text('Cerrar sesión',
                style: TextStyle(color: AppColors.error, fontWeight: FontWeight.w600)),
            onTap: () async {
              Navigator.pop(context);
              await ref.read(authProvider.notifier).logout();
              if (context.mounted) context.go('/login');
            },
          ),
        ],
      ),
      body: child,
    );
  }
}
```

---

## 7.2 DashboardProvider — `presentation/providers/dashboard_provider.dart`

```dart
// lib/presentation/providers/dashboard_provider.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../data/remote/api/category_remote_datasource.dart';
import '../../data/remote/api/product_remote_datasource.dart';
import '../../data/remote/api/order_remote_datasource.dart';
import '../../data/remote/api/user_remote_datasource.dart';
import '../../domain/model/product.dart';

class DashboardData {
  final int totalActiveProducts;
  final int outOfStockProducts;
  final int totalStock;
  final double avgPrice;
  final int activeCategories;
  final int totalCategories;
  final int totalOrders;
  final double totalRevenue;
  final int pendingOrders;
  final Map<String, int> ordersByStatus;
  final int activeUsers;
  final int totalUsers;
  final int staffUsers;
  final List<Product> lowStockProducts;

  const DashboardData({
    this.totalActiveProducts = 0,
    this.outOfStockProducts = 0,
    this.totalStock = 0,
    this.avgPrice = 0,
    this.activeCategories = 0,
    this.totalCategories = 0,
    this.totalOrders = 0,
    this.totalRevenue = 0,
    this.pendingOrders = 0,
    this.ordersByStatus = const {},
    this.activeUsers = 0,
    this.totalUsers = 0,
    this.staffUsers = 0,
    this.lowStockProducts = const [],
  });
}

sealed class DashboardState {
  const DashboardState();
}

class DashboardLoading extends DashboardState {
  const DashboardLoading();
}

class DashboardSuccess extends DashboardState {
  final DashboardData data;
  final DateTime loadedAt;
  const DashboardSuccess(this.data, this.loadedAt);
}

class DashboardError extends DashboardState {
  final String message;
  const DashboardError(this.message);
}

class DashboardNotifier extends StateNotifier<DashboardState> {
  final CategoryRemoteDatasource _catDs;
  final ProductRemoteDatasource _prodDs;
  final OrderRemoteDatasource _orderDs;
  final UserRemoteDatasource _userDs;

  DashboardNotifier(this._catDs, this._prodDs, this._orderDs, this._userDs)
      : super(const DashboardLoading()) {
    load();
  }

  Future<void> load() async {
    state = const DashboardLoading();
    try {
      // All calls in parallel with Future.wait
      final results = await Future.wait([
        _prodDs.getStats(),
        _catDs.getStats(),
        _orderDs.getStats(),
        _userDs.getStats(),
        _prodDs.getProducts(isActive: true, ordering: 'stock', pageSize: 10),
      ]);

      final prodStats = results[0] as Map<String, dynamic>;
      final catStats = results[1] as Map<String, dynamic>;
      final orderStats = results[2] as Map<String, dynamic>;
      final userStats = results[3] as Map<String, dynamic>;
      final lowStock = (results[4] as PaginatedProducts)
          .results
          .where((p) => p.stock < 5)
          .take(5)
          .toList();

      final byStatus = (orderStats['by_status'] as Map<String, dynamic>?)
              ?.map((k, v) => MapEntry(k, (v as num).toInt())) ??
          {};

      state = DashboardSuccess(
        DashboardData(
          totalActiveProducts: (prodStats['total_active'] as num?)?.toInt() ?? 0,
          outOfStockProducts: (prodStats['out_of_stock'] as num?)?.toInt() ?? 0,
          totalStock: (prodStats['total_stock'] as num?)?.toInt() ?? 0,
          avgPrice: (prodStats['avg_price'] as num?)?.toDouble() ?? 0,
          activeCategories: (catStats['active'] as num?)?.toInt() ?? 0,
          totalCategories: (catStats['total'] as num?)?.toInt() ?? 0,
          totalOrders: (orderStats['total_orders'] as num?)?.toInt() ?? 0,
          totalRevenue: (orderStats['total_revenue'] as num?)?.toDouble() ?? 0,
          pendingOrders: byStatus['pending'] ?? 0,
          ordersByStatus: byStatus,
          activeUsers: (userStats['active'] as num?)?.toInt() ?? 0,
          totalUsers: (userStats['total'] as num?)?.toInt() ?? 0,
          staffUsers: (userStats['staff'] as num?)?.toInt() ?? 0,
          lowStockProducts: lowStock,
        ),
        DateTime.now(),
      );
    } catch (e) {
      state = DashboardError(e.toString().replaceAll('Exception: ', ''));
    }
  }
}

final dashboardProvider =
    StateNotifierProvider<DashboardNotifier, DashboardState>((ref) {
  return DashboardNotifier(
    ref.watch(categoryDatasourceProvider),
    ref.watch(productDatasourceProvider),
    ref.watch(orderDatasourceProvider),
    ref.watch(userDatasourceProvider),
  );
});
```

---

## 7.3 KpiCard — `presentation/widgets/kpi_card.dart`

```dart
// lib/presentation/widgets/kpi_card.dart

import 'package:flutter/material.dart';
import '../../../theme/app_colors.dart';

class KpiCard extends StatelessWidget {
  final String       title;
  final String       value;
  final String?      subtitle;
  final IconData     icon;
  final Color        color;
  final bool         hasAlert;
  final VoidCallback? onTap;

  const KpiCard({
    super.key,
    required this.title,
    required this.value,
    required this.icon,
    this.subtitle,
    this.color    = AppColors.accent,
    this.hasAlert = false,
    this.onTap,
  });

  @override
  Widget build(BuildContext context) {
    final tt = Theme.of(context).textTheme;

    Widget card = Container(
      padding:    const EdgeInsets.all(14),
      decoration: BoxDecoration(
        color:        AppColors.surface,
        borderRadius: BorderRadius.circular(16),
        border:       hasAlert
            ? Border.all(color: AppColors.error.withValues(alpha: 0.3))
            : Border.all(color: AppColors.border),
      ),
      child: Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Container(
                width:  38, height: 38,
                decoration: BoxDecoration(
                  color:        color.withValues(alpha: 0.12),
                  borderRadius: BorderRadius.circular(10),
                ),
                child: Icon(icon, color: color, size: 20),
              ),
              if (hasAlert)
                const Text('⚠️', style: TextStyle(fontSize: 16)),
            ],
          ),
          const SizedBox(height: 12),
          Text(
            value,
            style: TextStyle(
              color:      AppColors.textPrimary,
              fontSize:   26,
              fontWeight: FontWeight.w800,
              height:     1,
            ),
          ),
          const SizedBox(height: 2),
          Text(title, style: tt.bodySmall),
          if (subtitle != null) ...[
            const SizedBox(height: 2),
            Text(
              subtitle!,
              style: TextStyle(
                color:    hasAlert ? AppColors.error : AppColors.textFaint,
                fontSize: 11,
              ),
            ),
          ],
        ],
      ),
    );

    return onTap != null
        ? GestureDetector(onTap: onTap, child: card)
        : card;
  }
}
```

---

## 7.4 DashboardScreen — `presentation/screens/admin/dashboard_screen.dart`

```dart
// lib/presentation/screens/admin/dashboard_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/formatters.dart';
import '../../../domain/model/order.dart';
import '../../providers/dashboard_provider.dart';
import '../../widgets/kpi_card.dart';

class DashboardScreen extends ConsumerWidget {
  const DashboardScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(dashboardProvider);

    return switch (state) {
      DashboardLoading() => const Center(
          child: CircularProgressIndicator(color: AppColors.accent),
        ),
      DashboardError(message: final msg) => Center(
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              const Text('⚠️', style: TextStyle(fontSize: 48)),
              const SizedBox(height: 12),
              Text(msg, style: const TextStyle(color: AppColors.error),
                  textAlign: TextAlign.center),
              const SizedBox(height: 16),
              ElevatedButton(
                onPressed: () => ref.read(dashboardProvider.notifier).load(),
                child:     const Text('Reintentar'),
              ),
            ],
          ),
        ),
      DashboardSuccess(data: final d, loadedAt: final loadedAt) =>
          _DashboardContent(data: d, loadedAt: loadedAt),
    };
  }
}

class _DashboardContent extends ConsumerWidget {
  final DashboardData data;
  final DateTime      loadedAt;
  const _DashboardContent({required this.data, required this.loadedAt});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final timeFmt = '${loadedAt.hour.toString().padLeft(2,'0')}:'
                    '${loadedAt.minute.toString().padLeft(2,'0')}:'
                    '${loadedAt.second.toString().padLeft(2,'0')}';

    return RefreshIndicator(
      color:       AppColors.accent,
      onRefresh:   () => ref.read(dashboardProvider.notifier).load(),
      child: ListView(
        padding: const EdgeInsets.all(16),
        children: [

          // ── Header ────────────────────────────────────────
          Row(
            mainAxisAlignment: MainAxisAlignment.spaceBetween,
            children: [
              Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text('Dashboard',
                      style: Theme.of(context).textTheme.headlineMedium),
                  Text('Actualizado: $timeFmt',
                      style: const TextStyle(color: AppColors.textFaint, fontSize: 11)),
                ],
              ),
              IconButton(
                icon:      const Icon(Icons.refresh_rounded, color: AppColors.accent),
                onPressed: () => ref.read(dashboardProvider.notifier).load(),
              ),
            ],
          ),
          const SizedBox(height: 20),

          // ── KPIs fila 1 ───────────────────────────────────
          Row(
            children: [
              Expanded(
                child: KpiCard(
                  title:    'Productos activos',
                  value:    '${data.totalActiveProducts}',
                  subtitle: data.outOfStockProducts > 0
                      ? '${data.outOfStockProducts} sin stock' : null,
                  icon:     Icons.inventory_2_outlined,
                  color:    AppColors.accent,
                  hasAlert: data.outOfStockProducts > 0,
                  onTap:    () => context.go('/admin/products'),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: KpiCard(
                  title:    'Categorías activas',
                  value:    '${data.activeCategories}',
                  subtitle: '${data.totalCategories} total',
                  icon:     Icons.category_outlined,
                  color:    AppColors.info,
                  onTap:    () => context.go('/admin/categories'),
                ),
              ),
            ],
          ),
          const SizedBox(height: 12),

          // ── KPIs fila 2 ───────────────────────────────────
          Row(
            children: [
              Expanded(
                child: KpiCard(
                  title:    'Pedidos totales',
                  value:    '${data.totalOrders}',
                  subtitle: data.pendingOrders > 0
                      ? '${data.pendingOrders} pendientes' : null,
                  icon:     Icons.shopping_bag_outlined,
                  color:    AppColors.success,
                  hasAlert: data.pendingOrders > 0,
                  onTap:    () => context.go('/admin/orders'),
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: KpiCard(
                  title:    'Usuarios activos',
                  value:    '${data.activeUsers}',
                  subtitle: '${data.totalUsers} registrados',
                  icon:     Icons.people_outline,
                  color:    AppColors.warning,
                  onTap:    () => context.go('/admin/users'),
                ),
              ),
            ],
          ),
          const SizedBox(height: 12),

          // ── KPIs fila 3 — financieros ─────────────────────
          Row(
            children: [
              Expanded(
                child: KpiCard(
                  title:  'Facturación total',
                  value:  formatPrice(data.totalRevenue),
                  icon:   Icons.trending_up_rounded,
                  color:  AppColors.accent,
                ),
              ),
              const SizedBox(width: 12),
              Expanded(
                child: KpiCard(
                  title:    'Precio medio',
                  value:    formatPrice(data.avgPrice),
                  subtitle: 'por producto',
                  icon:     Icons.sell_outlined,
                  color:    AppColors.textSecondary,
                ),
              ),
            ],
          ),
          const SizedBox(height: 20),

          // ── Pedidos por estado — barras ───────────────────
          if (data.ordersByStatus.isNotEmpty) ...[
            _SectionCard(
              title:     'Pedidos por estado',
              actionLabel: 'Ver todos',
              onAction:  () => context.go('/admin/orders'),
              child: Column(
                children: data.ordersByStatus.entries.map((entry) {
                  final status  = OrderStatus.fromValue(entry.key);
                  final count   = entry.value;
                  final total   = data.totalOrders.clamp(1, 999999);
                  final pct     = (count / total).clamp(0.02, 1.0);
                  final color   = orderStatusColor(status);

                  return Padding(
                    padding: const EdgeInsets.only(bottom: 10),
                    child: Column(
                      children: [
                        Row(
                          mainAxisAlignment: MainAxisAlignment.spaceBetween,
                          children: [
                            Text(status.label,
                                style: const TextStyle(
                                  color: AppColors.textSecondary, fontSize: 12,
                                )),
                            Text(
                              '$count',
                              style: TextStyle(
                                color: color, fontWeight: FontWeight.bold, fontSize: 12,
                              ),
                            ),
                          ],
                        ),
                        const SizedBox(height: 4),
                        ClipRRect(
                          borderRadius: BorderRadius.circular(3),
                          child: LinearProgressIndicator(
                            value:     pct,
                            minHeight: 7,
                            backgroundColor:  AppColors.surface2,
                            valueColor: AlwaysStoppedAnimation(color),
                          ),
                        ),
                      ],
                    ),
                  );
                }).toList(),
              ),
            ),
            const SizedBox(height: 14),
          ],

          // ── Stock bajo ────────────────────────────────────
          _SectionCard(
            title:      'Stock bajo',
            titleIcon:  Icons.warning_amber_rounded,
            titleColor: AppColors.warning,
            actionLabel:'Gestionar',
            onAction:   () => context.go('/admin/products'),
            child: data.lowStockProducts.isEmpty
                ? const Center(
                    child: Padding(
                      padding: EdgeInsets.symmetric(vertical: 8),
                      child:   Text(
                        '✅ Sin alertas de stock',
                        style: TextStyle(color: AppColors.success, fontSize: 13),
                      ),
                    ),
                  )
                : Column(
                    children: data.lowStockProducts.map((p) {
                      final isOut = p.stock == 0;
                      return GestureDetector(
                        onTap: () => context.go('/admin/products'),
                        child: Container(
                          margin:  const EdgeInsets.only(bottom: 6),
                          padding: const EdgeInsets.symmetric(
                            horizontal: 12, vertical: 10,
                          ),
                          decoration: BoxDecoration(
                            color:        AppColors.surface2,
                            borderRadius: BorderRadius.circular(10),
                          ),
                          child: Row(
                            mainAxisAlignment: MainAxisAlignment.spaceBetween,
                            children: [
                              Expanded(
                                child: Text(
                                  p.name,
                                  style: const TextStyle(
                                    color:    AppColors.textPrimary,
                                    fontSize: 13,
                                  ),
                                  overflow: TextOverflow.ellipsis,
                                ),
                              ),
                              Container(
                                padding: const EdgeInsets.symmetric(
                                  horizontal: 8, vertical: 2,
                                ),
                                decoration: BoxDecoration(
                                  color:        isOut
                                      ? AppColors.error.withValues(alpha: 0.12)
                                      : AppColors.warning.withValues(alpha: 0.12),
                                  borderRadius: BorderRadius.circular(6),
                                ),
                                child: Text(
                                  isOut ? 'Agotado' : '${p.stock} uds.',
                                  style: TextStyle(
                                    color:      isOut ? AppColors.error : AppColors.warning,
                                    fontSize:   11,
                                    fontWeight: FontWeight.bold,
                                  ),
                                ),
                              ),
                            ],
                          ),
                        ),
                      );
                    }).toList(),
                  ),
          ),
          const SizedBox(height: 14),

          // ── Acciones rápidas ──────────────────────────────
          _SectionCard(
            title: '⚡ Acciones rápidas',
            child: Wrap(
              spacing: 8, runSpacing: 8,
              children: [
                ('+ Categoría', AppColors.info,    '/admin/categories'),
                ('+ Producto',  AppColors.accent,  '/admin/products'),
                ('Ver pedidos', AppColors.success,  '/admin/orders'),
                ('Usuarios',    AppColors.warning,  '/admin/users'),
              ].map((item) => GestureDetector(
                onTap: () => context.go(item.$3),
                child: Container(
                  padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 8),
                  decoration: BoxDecoration(
                    color:        item.$2.withValues(alpha: 0.1),
                    borderRadius: BorderRadius.circular(10),
                    border:       Border.all(color: item.$2.withValues(alpha: 0.3)),
                  ),
                  child: Text(
                    item.$1,
                    style: TextStyle(
                      color:      item.$2,
                      fontWeight: FontWeight.bold,
                      fontSize:   13,
                    ),
                  ),
                ),
              )).toList(),
            ),
          ),
          const SizedBox(height: 32),
        ],
      ),
    );
  }
}

class _SectionCard extends StatelessWidget {
  final String       title;
  final IconData?    titleIcon;
  final Color?       titleColor;
  final String?      actionLabel;
  final VoidCallback? onAction;
  final Widget       child;

  const _SectionCard({
    required this.title,
    required this.child,
    this.titleIcon,
    this.titleColor,
    this.actionLabel,
    this.onAction,
  });

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
        Row(
          mainAxisAlignment: MainAxisAlignment.spaceBetween,
          children: [
            Row(
              children: [
                if (titleIcon != null) ...[
                  Icon(titleIcon, color: titleColor ?? AppColors.textPrimary, size: 18),
                  const SizedBox(width: 6),
                ],
                Text(
                  title,
                  style: TextStyle(
                    color:      titleColor ?? AppColors.textPrimary,
                    fontWeight: FontWeight.bold,
                    fontSize:   15,
                  ),
                ),
              ],
            ),
            if (actionLabel != null && onAction != null)
              GestureDetector(
                onTap: onAction,
                child: Text(
                  '$actionLabel →',
                  style: const TextStyle(
                    color: AppColors.accent, fontSize: 12, fontWeight: FontWeight.w600,
                  ),
                ),
              ),
          ],
        ),
        const SizedBox(height: 16),
        child,
      ],
    ),
  );
}
```

---

## 7.5 Actualizar el router — rutas admin con AdminShell

```dart
// lib/presentation/navigation/app_router.dart — reemplazar todas las rutas de admin

// Importar:
import '../screens/admin/dashboard_screen.dart';
import '../widgets/admin_shell.dart';

// Reemplazar rutas de admin:
GoRoute(
  path:    '/admin',
  builder: (_, state) => AdminShell(
    title:        'Dashboard',
    currentRoute: state.matchedLocation,
    child:        const DashboardScreen(),
  ),
),
GoRoute(
  path:    '/admin/categories',
  builder: (_, state) => AdminShell(
    title:        'Categorías',
    currentRoute: state.matchedLocation,
    child:        const _PlaceholderScreen('Categorías — M8'),
  ),
),
GoRoute(
  path:    '/admin/products',
  builder: (_, state) => AdminShell(
    title:        'Productos',
    currentRoute: state.matchedLocation,
    child:        const _PlaceholderScreen('Productos — M9'),
  ),
),
GoRoute(
  path:    '/admin/orders',
  builder: (_, state) => AdminShell(
    title:        'Pedidos',
    currentRoute: state.matchedLocation,
    child:        const _PlaceholderScreen('Pedidos admin — M10'),
  ),
),
GoRoute(
  path:    '/admin/orders/:id',
  builder: (_, state) => AdminShell(
    title:        'Detalle pedido',
    currentRoute: '/admin/orders',
    child:        _PlaceholderScreen('Pedido #${state.pathParameters['id']} — M10'),
  ),
),
GoRoute(
  path:    '/admin/users',
  builder: (_, state) => AdminShell(
    title:        'Usuarios',
    currentRoute: state.matchedLocation,
    child:        const _PlaceholderScreen('Usuarios — M11'),
  ),
),
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
        name='Agotado', defaults={'price': 30, 'stock': 0, 'category': cat, 'is_active': True}
    )
    Product.objects.get_or_create(
        name='Stock bajo', defaults={'price': 50, 'stock': 2, 'category': cat, 'is_active': True}
    )
print('Datos listos')
"
```

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Login con cuenta staff | Navega al Dashboard |
| 2 | Dashboard carga | 6 KPIs con datos reales del backend |
| 3 | KPI sin stock | Alerta ⚠️ y borde naranja-rojo |
| 4 | Tap en KPI | Navega a la sección correspondiente |
| 5 | `LinearProgressIndicator` por estado | Barras proporcionales con colores |
| 6 | Stock bajo | Productos con stock < 5 |
| 7 | Pull-to-refresh | Recarga todos los datos |
| 8 | Tap ☰ → Drawer | Panel con 5 secciones + avatar |
| 9 | Navegar a "Categorías" | Placeholder M8 dentro del AdminShell |
| 10 | Botón "← Tienda" | Navega a Home |

---

## Resumen

| Elemento | Estado |
|---|---|
| `AdminShell` con `NavigationDrawer` | ✅ |
| Header del drawer con avatar dorado | ✅ |
| `DashboardNotifier` con `Future.wait` paralelo | ✅ |
| `DashboardState` sealed class (Loading/Success/Error) | ✅ |
| `KpiCard` clickeable con alerta | ✅ |
| `DashboardScreen` con 6 KPIs en 3 filas | ✅ |
| `LinearProgressIndicator` por estado de pedido | ✅ |
| Lista de productos con stock bajo | ✅ |
| Acciones rápidas con `Wrap` | ✅ |
| Pull-to-refresh con `RefreshIndicator` | ✅ |
| Rutas admin con `AdminShell` contextual | ✅ |

**Siguiente módulo →** M8: Admin CRUD Categorías — lista, crear, editar y toggle optimista
