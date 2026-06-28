# Flutter Shop App — Módulo 4
## GoRouter completo — ShellRoute, BottomNavBar, Home y Catálogo

> **Objetivo:** Navegación definitiva con `ShellRoute` para la zona pública (con BottomNavBar), zona privada del cliente y zona admin. Home con hero y novedades. Catálogo con filtros, búsqueda debounced y paginación.
> **Checkpoint final:** navegar entre Home y Catálogo con la BottomNavBar, filtrar por categoría y buscar productos en tiempo real.

---

## 4.1 CartNotifier provisional — `presentation/providers/cart_provider.dart`

Necesario para el badge del carrito en la BottomNavBar antes del M6.

```dart
// lib/presentation/providers/cart_provider.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../domain/model/product.dart';

class CartItem {
  final Product product;
  final int     quantity;
  const CartItem({required this.product, required this.quantity});

  CartItem copyWith({int? quantity}) =>
      CartItem(product: product, quantity: quantity ?? this.quantity);

  double get subtotal => product.price * quantity;
}

class CartState {
  final List<CartItem> items;
  const CartState({this.items = const []});

  int    get totalItems   => items.fold(0, (s, i) => s + i.quantity);
  double get subtotal     => items.fold(0.0, (s, i) => s + i.subtotal);
  double get totalWithTax => items.fold(0.0, (s, i) => s + i.product.priceWithTax * i.quantity);

  CartState copyWith({List<CartItem>? items}) => CartState(items: items ?? this.items);
}

class CartNotifier extends StateNotifier<CartState> {
  CartNotifier() : super(const CartState());

  void addItem(Product product, {int quantity = 1}) {
    final idx = state.items.indexWhere((i) => i.product.id == product.id);
    if (idx >= 0) {
      final updated = List<CartItem>.from(state.items);
      final newQty  = (updated[idx].quantity + quantity).clamp(1, product.stock);
      updated[idx]  = updated[idx].copyWith(quantity: newQty);
      state = state.copyWith(items: updated);
    } else {
      state = state.copyWith(items: [...state.items, CartItem(product: product, quantity: quantity)]);
    }
  }

  void updateQuantity(int productId, int quantity) {
    if (quantity <= 0) {
      removeItem(productId);
      return;
    }
    state = state.copyWith(
      items: state.items.map((i) =>
        i.product.id == productId ? i.copyWith(quantity: quantity) : i,
      ).toList(),
    );
  }

  void removeItem(int productId) {
    state = state.copyWith(
      items: state.items.where((i) => i.product.id != productId).toList(),
    );
  }

  void clearCart() => state = const CartState();
}

final cartProvider = StateNotifierProvider<CartNotifier, CartState>(
  (_) => CartNotifier(),
);
```

---

## 4.2 Provider del catálogo — `presentation/providers/catalog_provider.dart`

```dart
// lib/presentation/providers/catalog_provider.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../data/remote/api/category_remote_datasource.dart';
import '../../data/remote/api/product_remote_datasource.dart';
import '../../data/repository/category_repository_impl.dart';
import '../../domain/model/category.dart';
import '../../domain/model/product.dart';

// ── Categorías ────────────────────────────────────────────────
final categoriesProvider = FutureProvider<List<Category>>((ref) {
  return ref.watch(categoryRepositoryProvider).getCategories();
});

// ── Estado del catálogo ───────────────────────────────────────
class CatalogState {
  final List<Product> products;
  final bool          isLoading;
  final bool          isLoadingMore;
  final String?       error;
  final int           total;
  final bool          hasMore;
  final String        search;
  final int?          selectedCategory;
  final String        ordering;
  final int           page;

  const CatalogState({
    this.products        = const [],
    this.isLoading       = false,
    this.isLoadingMore   = false,
    this.error,
    this.total           = 0,
    this.hasMore         = false,
    this.search          = '',
    this.selectedCategory,
    this.ordering        = '',
    this.page            = 1,
  });

  CatalogState copyWith({
    List<Product>? products,
    bool?          isLoading,
    bool?          isLoadingMore,
    String?        error,
    int?           total,
    bool?          hasMore,
    String?        search,
    int?           selectedCategory,
    bool           clearCategory = false,
    String?        ordering,
    int?           page,
  }) => CatalogState(
    products:         products         ?? this.products,
    isLoading:        isLoading        ?? this.isLoading,
    isLoadingMore:    isLoadingMore    ?? this.isLoadingMore,
    error:            error,
    total:            total            ?? this.total,
    hasMore:          hasMore          ?? this.hasMore,
    search:           search           ?? this.search,
    selectedCategory: clearCategory ? null : (selectedCategory ?? this.selectedCategory),
    ordering:         ordering         ?? this.ordering,
    page:             page             ?? this.page,
  );
}

class CatalogNotifier extends StateNotifier<CatalogState> {
  final ProductRemoteDatasource _datasource;
  CatalogNotifier(this._datasource) : super(const CatalogState()) {
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
      final result = await _datasource.getProducts(
        search:   s.search.isEmpty    ? null : s.search,
        category: s.selectedCategory,
        ordering: s.ordering.isEmpty  ? null : s.ordering,
        isActive: true,
        page:     page,
        pageSize: 12,
      );
      state = state.copyWith(
        products:     reset ? result.results : [...state.products, ...result.results],
        total:        result.count,
        hasMore:      result.next != null,
        isLoading:    false,
        isLoadingMore:false,
        page:         page + 1,
        error:        null,
      );
    } catch (e) {
      state = state.copyWith(
        isLoading:    false,
        isLoadingMore:false,
        error:        e.toString(),
      );
    }
  }

  void setSearch(String query) {
    state = state.copyWith(search: query);
    load();
  }

  void setCategory(int? id) {
    state = id == null
        ? state.copyWith(clearCategory: true)
        : state.copyWith(selectedCategory: id);
    load();
  }

  void setOrdering(String ordering) {
    state = state.copyWith(ordering: ordering);
    load();
  }

  void loadMore() => load(reset: false);
  void refresh()  => load();
}

final catalogProvider = StateNotifierProvider<CatalogNotifier, CatalogState>((ref) {
  return CatalogNotifier(ref.watch(productDatasourceProvider));
});

// Provider de un producto individual
final productDetailProvider = FutureProvider.family<Product, int>((ref, id) {
  return ref.watch(productDatasourceProvider).getProduct(id);
});
```

---

## 4.3 Shell de la zona pública — `presentation/navigation/public_shell.dart`

```dart
// lib/presentation/navigation/public_shell.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../providers/cart_provider.dart';
import '../../theme/app_colors.dart';

class PublicShell extends ConsumerWidget {
  final Widget child;
  final bool   showCart;
  const PublicShell({super.key, required this.child, this.showCart = true});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cartCount   = ref.watch(cartProvider).totalItems;
    final location    = GoRouterState.of(context).matchedLocation;

    int _selectedIndex() {
      if (location.startsWith('/catalog')) return 1;
      if (location.startsWith('/orders'))  return 2;
      if (location.startsWith('/profile')) return 3;
      return 0;
    }

    return Scaffold(
      body: child,
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _selectedIndex(),
        type:         BottomNavigationBarType.fixed,
        items: [
          const BottomNavigationBarItem(
            icon:  Icon(Icons.home_outlined),
            activeIcon: Icon(Icons.home),
            label: 'Inicio',
          ),
          const BottomNavigationBarItem(
            icon:  Icon(Icons.grid_view_outlined),
            activeIcon: Icon(Icons.grid_view),
            label: 'Catálogo',
          ),
          BottomNavigationBarItem(
            icon: Stack(
              clipBehavior: Clip.none,
              children: [
                const Icon(Icons.shopping_cart_outlined),
                if (cartCount > 0)
                  Positioned(
                    right: -6,
                    top:   -4,
                    child: Container(
                      padding:    const EdgeInsets.all(3),
                      decoration: const BoxDecoration(
                        color: AppColors.error,
                        shape: BoxShape.circle,
                      ),
                      child: Text(
                        cartCount > 99 ? '99+' : cartCount.toString(),
                        style: const TextStyle(
                          color:    Colors.white,
                          fontSize: 9,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                    ),
                  ),
              ],
            ),
            activeIcon: const Icon(Icons.shopping_cart),
            label: 'Carrito',
          ),
          const BottomNavigationBarItem(
            icon:  Icon(Icons.person_outline),
            activeIcon: Icon(Icons.person),
            label: 'Perfil',
          ),
        ],
        onTap: (index) {
          switch (index) {
            case 0: context.go('/');        break;
            case 1: context.go('/catalog'); break;
            case 2: context.go('/cart');    break;
            case 3: context.go('/profile'); break;
          }
        },
      ),
    );
  }
}
```

---

## 4.4 Widget ProductCard — `presentation/widgets/product_card.dart`

```dart
// lib/presentation/widgets/product_card.dart

import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';
import '../../theme/app_colors.dart';
import '../../core/utils/formatters.dart';
import '../../domain/model/product.dart';

class ProductCard extends StatelessWidget {
  final Product      product;
  final VoidCallback onTap;

  const ProductCard({super.key, required this.product, required this.onTap});

  @override
  Widget build(BuildContext context) {
    final tt = Theme.of(context).textTheme;

    return GestureDetector(
      onTap: onTap,
      child: Container(
        decoration: BoxDecoration(
          color:        AppColors.surface,
          borderRadius: BorderRadius.circular(16),
          border:       Border.all(color: AppColors.border),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          mainAxisSize: MainAxisSize.min,
          children: [
            // Imagen
            ClipRRect(
              borderRadius: const BorderRadius.vertical(top: Radius.circular(16)),
              child: AspectRatio(
                aspectRatio: 1,
                child: product.imageUrl != null
                    ? CachedNetworkImage(
                        imageUrl:   product.imageUrl!,
                        fit:        BoxFit.cover,
                        placeholder:(_, __) => Container(color: AppColors.surface2),
                        errorWidget:(_, __, ___) => _ImagePlaceholder(),
                      )
                    : _ImagePlaceholder(),
              ),
            ),

            // Info
            Expanded(
              child: Padding(
                padding: const EdgeInsets.all(8),
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  mainAxisSize: MainAxisSize.min,
                  children: [
                    // Categoría
                    if (product.category != null)
                      Text(
                        product.category!.name.toUpperCase(),
                        style: const TextStyle(
                          color:         AppColors.accent,
                          fontSize:      9,
                          fontWeight:    FontWeight.bold,
                          letterSpacing: 0.5,
                        ),
                        maxLines: 1,
                        overflow: TextOverflow.ellipsis,
                      ),
                    const SizedBox(height: 2),

                    // Nombre
                    Flexible(
                      child: Text(
                        product.name,
                        style:    tt.bodySmall?.copyWith(
                          color:      AppColors.textPrimary,
                          fontWeight: FontWeight.w600,
                        ),
                        maxLines: 1,
                        overflow: TextOverflow.ellipsis,
                      ),
                    ),
                    const SizedBox(height: 4),

                    // Precio
                    Text(
                      formatPrice(product.price),
                      style: const TextStyle(
                        color:      AppColors.accent,
                        fontWeight: FontWeight.bold,
                        fontSize:   11,
                      ),
                    ),
                    const SizedBox(height: 2),
                    Text(
                      '${formatPrice(product.priceWithTax)} IVA',
                      style: const TextStyle(
                        color:    AppColors.textSecondary,
                        fontSize: 8,
                      ),
                      maxLines: 1,
                      overflow: TextOverflow.ellipsis,
                    ),

                    // Sin stock
                    if (!product.inStock) ...[
                      const SizedBox(height: 4),
                      Container(
                        padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 1),
                        decoration: BoxDecoration(
                          color:        AppColors.error.withValues(alpha: 0.12),
                          borderRadius: BorderRadius.circular(3),
                        ),
                        child: const Text(
                          'Sin stock',
                          style: TextStyle(
                            color:     AppColors.error,
                            fontSize:  8,
                            fontWeight:FontWeight.bold,
                          ),
                        ),
                      ),
                    ],
                  ],
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _ImagePlaceholder extends StatelessWidget {
  @override
  Widget build(BuildContext context) => Container(
    color:      AppColors.surface2,
    alignment:  Alignment.center,
    child:      const Text('📦', style: TextStyle(fontSize: 40)),
  );
}
```

---

## 4.5 Pantalla Home — `presentation/screens/catalog/home_screen.dart`

```dart
// lib/presentation/screens/catalog/home_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../providers/catalog_provider.dart';
import '../../widgets/product_card.dart';

class HomeScreen extends ConsumerWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final categoriesAsync = ref.watch(categoriesProvider);
    final catalogState    = ref.watch(catalogProvider);
    final tt              = Theme.of(context).textTheme;

    return Scaffold(
      body: CustomScrollView(
        slivers: [
          // ── Hero ─────────────────────────────────────────
          SliverToBoxAdapter(
            child: Container(
              width:  double.infinity,
              padding: const EdgeInsets.fromLTRB(24, 72, 24, 48),
              decoration: BoxDecoration(
                gradient: LinearGradient(
                  begin: Alignment.topCenter,
                  end:   Alignment.bottomCenter,
                  colors: [AppColors.surface2, AppColors.background],
                ),
              ),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    'Descubre lo',
                    style: tt.headlineLarge?.copyWith(
                      color:      AppColors.textSecondary,
                      fontWeight: FontWeight.w300,
                    ),
                  ),
                  Text(
                    'extraordinario',
                    style: tt.displaySmall?.copyWith(
                      color:      AppColors.accent,
                      fontWeight: FontWeight.bold,
                    ),
                  ),
                  const SizedBox(height: 12),
                  Text(
                    'Los mejores productos seleccionados para ti.',
                    style: tt.bodyMedium,
                  ),
                  const SizedBox(height: 24),
                  FilledButton.icon(
                    onPressed: () => context.go('/catalog'),
                    icon:      const Icon(Icons.grid_view_rounded, size: 18),
                    label:     const Text('Ver catálogo'),
                    style:     FilledButton.styleFrom(
                      backgroundColor: AppColors.accent,
                      foregroundColor: AppColors.onAccent,
                      shape: RoundedRectangleBorder(
                        borderRadius: BorderRadius.circular(10),
                      ),
                    ),
                  ),
                ],
              ),
            ),
          ),

          // ── Categorías ────────────────────────────────────
          SliverToBoxAdapter(
            child: categoriesAsync.when(
              loading: () => const SizedBox.shrink(),
              error:   (_, __) => const SizedBox.shrink(),
              data: (cats) {
                final active = cats.where((c) => c.isActive).take(6).toList();
                if (active.isEmpty) return const SizedBox.shrink();
                return Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Padding(
                      padding: const EdgeInsets.fromLTRB(24, 0, 24, 12),
                      child: Row(
                        mainAxisAlignment: MainAxisAlignment.spaceBetween,
                        children: [
                          Text('Categorías', style: tt.titleLarge),
                          TextButton(
                            onPressed: () => context.go('/catalog'),
                            child: const Text('Ver todas'),
                          ),
                        ],
                      ),
                    ),
                    SizedBox(
                      height: 80,
                      child:  ListView.separated(
                        padding:          const EdgeInsets.symmetric(horizontal: 24),
                        scrollDirection:  Axis.horizontal,
                        itemCount:        active.length,
                        separatorBuilder: (_, __) => const SizedBox(width: 10),
                        itemBuilder: (_, i) {
                          final cat = active[i];
                          return GestureDetector(
                            onTap: () {
                              ref.read(catalogProvider.notifier).setCategory(cat.id);
                              context.go('/catalog');
                            },
                            child: Container(
                              width:   110,
                              padding: const EdgeInsets.all(10),
                              decoration: BoxDecoration(
                                color:        AppColors.surface,
                                borderRadius: BorderRadius.circular(12),
                                border:       Border.all(color: AppColors.border),
                              ),
                              child: Column(
                                mainAxisAlignment: MainAxisAlignment.center,
                                children: [
                                  const Text('🏷️', style: TextStyle(fontSize: 22)),
                                  const SizedBox(height: 4),
                                  Text(
                                    cat.name,
                                    style:    const TextStyle(
                                      color:     AppColors.textPrimary,
                                      fontSize:  11,
                                      fontWeight:FontWeight.w600,
                                    ),
                                    maxLines:  1,
                                    overflow:  TextOverflow.ellipsis,
                                    textAlign: TextAlign.center,
                                  ),
                                ],
                              ),
                            ),
                          );
                        },
                      ),
                    ),
                    const SizedBox(height: 24),
                  ],
                );
              },
            ),
          ),

          // ── Novedades — encabezado ─────────────────────────
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.fromLTRB(24, 0, 24, 16),
              child:   Row(
                mainAxisAlignment: MainAxisAlignment.spaceBetween,
                children: [
                  Text('Novedades', style: tt.titleLarge),
                  TextButton(
                    onPressed: () => context.go('/catalog'),
                    child: const Text('Ver todos'),
                  ),
                ],
              ),
            ),
          ),

          // ── Novedades — grid ───────────────────────────────
          if (catalogState.isLoading && catalogState.products.isEmpty)
            const SliverToBoxAdapter(
              child: Center(
                child: Padding(
                  padding: EdgeInsets.all(32),
                  child:   CircularProgressIndicator(color: AppColors.accent),
                ),
              ),
            )
          else
            SliverPadding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              sliver: SliverGrid(
                delegate: SliverChildBuilderDelegate(
                  (ctx, i) {
                    final product = catalogState.products.take(4).toList()[i];
                    return ProductCard(
                      product: product,
                      onTap:   () => context.push('/product/${product.id}'),
                    );
                  },
                  childCount: catalogState.products.take(4).length,
                ),
                gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                  crossAxisCount:    2,
                  crossAxisSpacing:  12,
                  mainAxisSpacing:   12,
                  childAspectRatio:  0.68,
                ),
              ),
            ),

          const SliverToBoxAdapter(child: SizedBox(height: 32)),
        ],
      ),
    );
  }
}
```

---

## 4.6 Pantalla Catálogo — `presentation/screens/catalog/catalog_screen.dart`

```dart
// lib/presentation/screens/catalog/catalog_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../providers/catalog_provider.dart';
import '../../widgets/product_card.dart';

class CatalogScreen extends ConsumerStatefulWidget {
  const CatalogScreen({super.key});

  @override
  ConsumerState<CatalogScreen> createState() => _CatalogScreenState();
}

class _CatalogScreenState extends ConsumerState<CatalogScreen> {
  final _searchCtrl  = TextEditingController();
  final _scrollCtrl  = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollCtrl.addListener(_onScroll);
  }

  @override
  void dispose() {
    _searchCtrl.dispose();
    _scrollCtrl.dispose();
    super.dispose();
  }

  void _onScroll() {
    if (_scrollCtrl.position.pixels >= _scrollCtrl.position.maxScrollExtent - 200) {
      ref.read(catalogProvider.notifier).loadMore();
    }
  }

  @override
  Widget build(BuildContext context) {
    final state        = ref.watch(catalogProvider);
    final catsAsync    = ref.watch(categoriesProvider);
    final tt           = Theme.of(context).textTheme;

    return Scaffold(
      body: SafeArea(
        child: Column(
          children: [
            // ── Header ──────────────────────────────────────
            Container(
              color:   AppColors.surface,
              padding: const EdgeInsets.fromLTRB(16, 16, 16, 0),
              child:   Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceBetween,
                    children: [
                      Text('Catálogo', style: tt.headlineMedium),
                      Text(
                        '${state.total} productos',
                        style: tt.bodySmall,
                      ),
                    ],
                  ),
                  const SizedBox(height: 12),

                  // Búsqueda
                  TextField(
                    controller:  _searchCtrl,
                    onChanged:   (q) {
                      ref.read(catalogProvider.notifier).setSearch(q);
                    },
                    decoration: const InputDecoration(
                      hintText:    'Buscar productos...',
                      prefixIcon:  Icon(Icons.search_rounded, color: AppColors.textSecondary),
                      contentPadding: EdgeInsets.symmetric(vertical: 12),
                    ),
                    style: const TextStyle(color: AppColors.textPrimary),
                  ),
                  const SizedBox(height: 10),

                  // Chips de ordenamiento
                  SizedBox(
                    height: 34,
                    child: ListView(
                      scrollDirection: Axis.horizontal,
                      children: [
                        for (final item in [
                          ('',             'Relevancia'),
                          ('price',        'Precio ↑'),
                          ('-price',       'Precio ↓'),
                          ('-created_at',  'Recientes'),
                        ])
                          Padding(
                            padding: const EdgeInsets.only(right: 8),
                            child:   ChoiceChip(
                              label:     Text(item.$2),
                              selected:  state.ordering == item.$1,
                              onSelected:(_) => ref.read(catalogProvider.notifier).setOrdering(item.$1),
                            ),
                          ),
                      ],
                    ),
                  ),
                  const SizedBox(height: 8),

                  // Chips de categorías
                  catsAsync.when(
                    loading: () => const SizedBox.shrink(),
                    error:   (_, __) => const SizedBox.shrink(),
                    data: (cats) => SizedBox(
                      height: 34,
                      child: ListView(
                        scrollDirection: Axis.horizontal,
                        children: [
                          Padding(
                            padding: const EdgeInsets.only(right: 8),
                            child:   ChoiceChip(
                              label:     const Text('Todas'),
                              selected:  state.selectedCategory == null,
                              onSelected:(_) => ref.read(catalogProvider.notifier).setCategory(null),
                            ),
                          ),
                          for (final cat in cats.where((c) => c.isActive))
                            Padding(
                              padding: const EdgeInsets.only(right: 8),
                              child:   ChoiceChip(
                                label:     Text(cat.name),
                                selected:  state.selectedCategory == cat.id,
                                onSelected:(_) =>
                                    ref.read(catalogProvider.notifier).setCategory(cat.id),
                              ),
                            ),
                        ],
                      ),
                    ),
                  ),
                  const SizedBox(height: 12),
                ],
              ),
            ),

            // ── Grid de productos ────────────────────────────
            Expanded(
              child: Builder(
                builder: (_) {
                  if (state.isLoading && state.products.isEmpty) {
                    return const Center(
                      child: CircularProgressIndicator(color: AppColors.accent),
                    );
                  }
                  if (state.error != null && state.products.isEmpty) {
                    return Center(
                      child: Column(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          const Text('❌', style: TextStyle(fontSize: 40)),
                          const SizedBox(height: 12),
                          Text(state.error!, style: const TextStyle(color: AppColors.error)),
                          const SizedBox(height: 16),
                          FilledButton(
                            onPressed: () => ref.read(catalogProvider.notifier).refresh(),
                            style:     FilledButton.styleFrom(
                              backgroundColor: AppColors.accent,
                              foregroundColor: AppColors.onAccent,
                            ),
                            child: const Text('Reintentar'),
                          ),
                        ],
                      ),
                    );
                  }
                  if (state.products.isEmpty) {
                    return const Center(
                      child: Column(
                        mainAxisAlignment: MainAxisAlignment.center,
                        children: [
                          Text('🔍', style: TextStyle(fontSize: 48)),
                          SizedBox(height: 12),
                          Text('Sin resultados', style: TextStyle(color: AppColors.textPrimary, fontSize: 18, fontWeight: FontWeight.bold)),
                          Text('Intenta con otra búsqueda', style: TextStyle(color: AppColors.textSecondary)),
                        ],
                      ),
                    );
                  }

                  return GridView.builder(
                    controller:  _scrollCtrl,
                    padding:     const EdgeInsets.all(16),
                    gridDelegate:const SliverGridDelegateWithFixedCrossAxisCount(
                      crossAxisCount:   2,
                      crossAxisSpacing: 12,
                      mainAxisSpacing:  12,
                      childAspectRatio: 0.68,
                    ),
                    itemCount:  state.products.length + (state.isLoadingMore ? 1 : 0),
                    itemBuilder:(ctx, i) {
                      if (i >= state.products.length) {
                        return const Center(
                          child: Padding(
                            padding: EdgeInsets.all(16),
                            child:   CircularProgressIndicator(
                              color:       AppColors.accent,
                              strokeWidth: 2,
                            ),
                          ),
                        );
                      }
                      final product = state.products[i];
                      return ProductCard(
                        product: product,
                        onTap:   () => context.push('/product/${product.id}'),
                      );
                    },
                  );
                },
              ),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## 4.7 Router completo — `presentation/navigation/app_router.dart`

```dart
// lib/presentation/navigation/app_router.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../domain/model/auth_state.dart';
import '../providers/auth_provider.dart';
import '../screens/auth/login_screen.dart';
import '../screens/auth/register_screen.dart';
import '../screens/catalog/catalog_screen.dart';
import '../screens/catalog/home_screen.dart';
import 'public_shell.dart';

class _PlaceholderScreen extends ConsumerWidget {
  final String title;
  const _PlaceholderScreen(this.title);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(
        title: Text(title),
        actions: [
          IconButton(
            tooltip: 'Cerrar sesión',
            icon: const Icon(Icons.logout),
            onPressed: () async {
              // Cerrar sesión y volver al login
              await ref.read(authProvider.notifier).logout();
              context.go('/login');
            },
          ),
        ],
      ),
      body: Center(
        child: Text(title, style: const TextStyle(color: Color(0xFF8888AA), fontSize: 16)),
      ),
    );
  }
}

final routerProvider = Provider<GoRouter>((ref) {
  return GoRouter(
    initialLocation: '/',
    refreshListenable: _AuthStateListenable(ref),
    redirect: (context, state) {
      final auth     = ref.read(authProvider);
      final location = state.matchedLocation;

      if (auth.isChecking)        return null;

      final isAuthRoute = location == '/login' || location == '/register';

      if (!auth.isAuthenticated && !isAuthRoute) return '/login';
      if ( auth.isAuthenticated &&  isAuthRoute) return auth.isStaff ? '/admin' : '/';
      if ( auth.isAuthenticated && !auth.isStaff && location.startsWith('/admin')) return '/';

      return null;
    },
    routes: [
      // ── Auth ──────────────────────────────────────────────
      GoRoute(path: '/login',    builder: (_, __) => const LoginScreen()),
      GoRoute(path: '/register', builder: (_, __) => const RegisterScreen()),

      // ── Zona pública con BottomNavBar ──────────────────────
      ShellRoute(
        builder: (_, __, child) => PublicShell(child: child),
        routes: [
          GoRoute(path: '/',        builder: (_, __) => const HomeScreen()),
          GoRoute(path: '/catalog', builder: (_, __) => const CatalogScreen()),
          GoRoute(
            path:    '/product/:id',
            builder: (_, s) => _PlaceholderScreen('Detalle #${s.pathParameters['id']} — M5'),
          ),
          GoRoute(path: '/cart',    builder: (_, __) => const _PlaceholderScreen('Carrito — M6')),
          GoRoute(path: '/orders',  builder: (_, __) => const _PlaceholderScreen('Mis pedidos — M7')),
          GoRoute(path: '/orders/:id', builder: (_, s) => _PlaceholderScreen('Pedido #${s.pathParameters['id']} — M7')),
          GoRoute(path: '/profile', builder: (_, __) => const _PlaceholderScreen('Perfil — M7')),
        ],
      ),

      // ── Admin ─────────────────────────────────────────────
      GoRoute(path: '/admin',              builder: (_, __) => const _PlaceholderScreen('Dashboard — M8')),
      GoRoute(path: '/admin/categories',   builder: (_, __) => const _PlaceholderScreen('Categorías — M9')),
      GoRoute(path: '/admin/products',     builder: (_, __) => const _PlaceholderScreen('Productos — M10')),
      GoRoute(path: '/admin/orders',       builder: (_, __) => const _PlaceholderScreen('Pedidos admin — M11')),
      GoRoute(path: '/admin/orders/:id',   builder: (_, s) => _PlaceholderScreen('Pedido admin #${s.pathParameters['id']} — M11')),
      GoRoute(path: '/admin/users',        builder: (_, __) => const _PlaceholderScreen('Usuarios — M12')),
    ],
  );
});

class _AuthStateListenable extends ChangeNotifier {
  _AuthStateListenable(Ref ref) {
    ref.listen<AuthState>(authProvider, (_, __) => notifyListeners());
  }
}
```

---

## ✅ Checkpoint Módulo 4

```bash
flutter run
```

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Login con cuenta cliente | Home con hero y novedades |
| 2 | Hero → "Ver catálogo" | Navega al catálogo |
| 3 | Chips de categoría en Home | Tap navega al catálogo filtrado |
| 4 | Grid en Home | 4 productos más recientes |
| 5 | Catálogo → buscar "laptop" | Filtrado con petición al backend |
| 6 | Chip "Precio ↑" | Productos ordenados por precio |
| 7 | Chip de categoría | Solo productos de esa categoría |
| 8 | Scroll al fondo del catálogo | Carga más productos automáticamente |
| 9 | BottomNavBar | 4 tabs: Inicio, Catálogo, Carrito, Perfil |
| 10 | Badge del carrito | Aparece con número de ítems (M6) |
| 11 | Tabs Carrito y Perfil | Placeholder "próximo módulo" |
| 12 | Login con staff | Navega a Dashboard placeholder |

---

## Resumen

| Elemento | Estado |
|---|---|
| `CartNotifier` provisional con StateNotifier | ✅ |
| `CatalogState` + `CatalogNotifier` con paginación | ✅ |
| `productDetailProvider` FutureProvider.family | ✅ |
| `PublicShell` con BottomNavBar y badge del carrito | ✅ |
| `ProductCard` con CachedNetworkImage | ✅ |
| `HomeScreen` con SliverList, hero, categorías y novedades | ✅ |
| `CatalogScreen` con búsqueda, chips y GridView | ✅ |
| GoRouter completo con `ShellRoute` | ✅ |
| Guards por rol en el redirect de GoRouter | ✅ |

**Siguiente módulo →** M5: Detalle de producto y carrito con DraggableScrollableSheet
