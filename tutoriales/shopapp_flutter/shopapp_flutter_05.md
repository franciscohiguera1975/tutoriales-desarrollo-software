# Flutter Shop App — Módulo 5
## Búsqueda, filtros y detalle de producto
### SearchBar con debounce, ModalBottomSheet de filtros y pantalla de detalle

---

## Objetivo del módulo

Al terminar este módulo tendrás:
- Barra de búsqueda en el Catálogo con debounce de 500 ms
- `ModalBottomSheet` de filtros: categoría, rango de precio y ordenamiento
- Pantalla de detalle del producto con precio IVA, stock, selector de cantidad
- Navegación desde la grilla al detalle con go_router

**Lo que puedes probar:** buscar "laptop", filtrar por categoría, ver el detalle con precio IVA calculado, cambiar la cantidad.

---

## 1. Actualizar CatalogNotifier para incluir categorías

El `CatalogNotifier` de M4 ya maneja productos. Ahora añadiremos categorías. Las categorías ya están disponibles desde `categoryDatasourceProvider` creado en M2.

```dart
// lib/presentation/providers/catalog_provider.dart — versión M5 actualizada

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../data/remote/api/category_remote_datasource.dart';
import '../../data/remote/api/product_remote_datasource.dart';
import '../../domain/model/category.dart';
import '../../domain/model/product.dart';

// ── Estado del catálogo ──────────────────────────────────────
class CatalogState {
  final List<Product> products;
  final List<Category> categories;
  final bool isLoading;
  final bool isLoadingMore;
  final String? error;
  final int total;
  final bool hasMore;
  final int page;
  final String? search;
  final int? categoryId;
  final double? minPrice;
  final double? maxPrice;
  final String? ordering;

  const CatalogState({
    this.products = const [],
    this.categories = const [],
    this.isLoading = false,
    this.isLoadingMore = false,
    this.error,
    this.total = 0,
    this.hasMore = false,
    this.page = 1,
    this.search,
    this.categoryId,
    this.minPrice,
    this.maxPrice,
    this.ordering,
  });

  CatalogState copyWith({
    List<Product>? products,
    List<Category>? categories,
    bool? isLoading,
    bool? isLoadingMore,
    String? error,
    int? total,
    bool? hasMore,
    int? page,
    String? search,
    int? categoryId,
    double? minPrice,
    double? maxPrice,
    String? ordering,
  }) => CatalogState(
    products: products ?? this.products,
    categories: categories ?? this.categories,
    isLoading: isLoading ?? this.isLoading,
    isLoadingMore: isLoadingMore ?? this.isLoadingMore,
    error: error,
    total: total ?? this.total,
    hasMore: hasMore ?? this.hasMore,
    page: page ?? this.page,
    search: search,
    // lib/presentation/screens/catalog/catalog_screen.dart — versión M5

    import 'package:flutter/material.dart' hide SearchBar;
    import 'package:flutter_riverpod/flutter_riverpod.dart';
    import 'package:go_router/go_router.dart';
    import '../../../theme/app_colors.dart';
    import '../../providers/catalog_provider.dart';
    import '../../widgets/product_card.dart';
    import 'package:flutter_shop_app/presentation/widgets/filters_sheet.dart';
    import '../../widgets/search_bar.dart';

    class CatalogScreen extends ConsumerStatefulWidget {
      const CatalogScreen({super.key});

      @override
      ConsumerState<CatalogScreen> createState() => _CatalogScreenState();
    }

    class _CatalogScreenState extends ConsumerState<CatalogScreen> {
      final _scrollController = ScrollController();

      @override
      void initState() {
        super.initState();
        _scrollController.addListener(() {
          if (_scrollController.position.pixels >=
              _scrollController.position.maxScrollExtent - 200) {
            ref.read(catalogProvider.notifier).loadMore();
          }
        });
      }

      @override
      void dispose() {
        _scrollController.dispose();
        super.dispose();
      }

      Future<void> _openFilters() async {
        final state = ref.read(catalogProvider);
        final activeFilters = ProductFilters(
          categoryId: state.categoryId,
          ordering: state.ordering,
          minPrice: state.minPrice,
          maxPrice: state.maxPrice,
        );
        final result = await showFiltersSheet(
          context: context,
          activeFilters: activeFilters,
          categories: state.categories,
        );
        if (result != null && mounted) {
          ref.read(catalogProvider.notifier).setCategory(result.categoryId);
          ref.read(catalogProvider.notifier).setOrdering(result.ordering);
          ref.read(catalogProvider.notifier).setPriceRange(result.minPrice, result.maxPrice);
        }
      }

      @override
      Widget build(BuildContext context) {
        final state = ref.watch(catalogProvider);
        final numFilters = _countActiveFilters(state);

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
                ElevatedButton(
                  onPressed: () => ref.read(catalogProvider.notifier).refresh(),
                  child: const Text('Retry'),
                ),
              ],
            ),
          );
        }

        return RefreshIndicator(
          color: AppColors.accent,
          onRefresh: ref.read(catalogProvider.notifier).refresh,
          child: CustomScrollView(
            controller: _scrollController,
            slivers: [
              // ── Search bar + filter button ─────────────
              SliverToBoxAdapter(
                child: Padding(
                  padding: const EdgeInsets.fromLTRB(16, 12, 16, 8),
                  child: Row(
                    children: [
                      Expanded(
                        child: SearchBar(
                          initialValue: state.search,
                          onChanged: (q) => ref.read(catalogProvider.notifier).setSearch(q),
                        ),
                      ),
                      const SizedBox(width: 8),
                      Stack(
                        children: [
                          IconButton.filled(
                            style: IconButton.styleFrom(
                              backgroundColor: numFilters > 0
                                  ? AppColors.accent
                                  : AppColors.surface,
                              foregroundColor: numFilters > 0
                                  ? AppColors.onAccent
                                  : AppColors.textPrimary,
                              shape: RoundedRectangleBorder(
                                borderRadius: BorderRadius.circular(12),
                                side: const BorderSide(color: AppColors.border),
                              ),
                            ),
                            icon: const Icon(Icons.tune),
                            onPressed: _openFilters,
                          ),
                          if (numFilters > 0)
                            Positioned(
                              top: 4,
                              right: 4,
                              child: Container(
                                width: 16,
                                height: 16,
                                decoration: const BoxDecoration(
                                  color: AppColors.error,
                                  shape: BoxShape.circle,
                                ),
                                child: Center(
                                  child: Text(
                                    '$numFilters',
                                    style: const TextStyle(
                                      color: Colors.white,
                                      fontSize: 10,
                                      fontWeight: FontWeight.w700,
                                    ),
                                  ),
                                ),
                              ),
                            ),
                        ],
                      ),
                    ],
                  ),
                ),
              ),

              // ── Results count ──────────────────────────────
              SliverToBoxAdapter(
                child: Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 16),
                  child: Text(
                    '${state.total} result${state.total != 1 ? 's' : ''}',
                    style: const TextStyle(fontSize: 13, color: AppColors.textSecondary),
                  ),
                ),
              ),

              // ── Grid ────────────────────────────────────────
              if (state.products.isEmpty && !state.isLoading)
                const SliverFillRemaining(
                  child: Center(
                    child: Column(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        Text('📦', style: TextStyle(fontSize: 52)),
                        SizedBox(height: 16),
                        Text('No results',
                            style: TextStyle(
                                color: AppColors.textPrimary,
                                fontSize: 18,
                                fontWeight: FontWeight.bold)),
                      ],
                    ),
                  ),
                )
              else
                SliverPadding(
                  padding: const EdgeInsets.all(16),
                  sliver: SliverGrid(
                    gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                      crossAxisCount: 2,
                      mainAxisSpacing: 8,
                      crossAxisSpacing: 8,
                      childAspectRatio: 0.72,
                    ),
                    delegate: SliverChildBuilderDelegate(
                      (_, i) {
                        final p = state.products[i];
                        return ProductCard(
                          product: p,
                          onTap: () => context.push('/catalog/${p.id}'),
                        );
                      },
                      childCount: state.products.length,
                    ),
                  ),
                ),

              // ── Loading more spinner ────────────────────────
              if (state.isLoadingMore)
                const SliverToBoxAdapter(
                  child: Padding(
                    padding: EdgeInsets.all(16),
                    child: Center(
                      child: CircularProgressIndicator(color: AppColors.accent),
                    ),
                  ),
                ),
              const SliverToBoxAdapter(child: SizedBox(height: 32)),
            ],
          ),
        );
      }

      int _countActiveFilters(CatalogState state) {
        int count = 0;
        if (state.categoryId != null) count++;
        if (state.ordering != null) count++;
        if (state.minPrice != null) count++;
        if (state.maxPrice != null) count++;
        return count;
      }
    }
      activeFilters: activeFilters,
      categories: categories,
    ),
  );
}

class _FiltersSheet extends StatefulWidget {
  final ProductFilters activeFilters;
  final List<Category> categories;

  const _FiltersSheet({required this.activeFilters, required this.categories});

  @override
  State<_FiltersSheet> createState() => _FiltersSheetState();
}

class _FiltersSheetState extends State<_FiltersSheet> {
  late int? _categoryId;
  late String? _ordering;
  final _ctrlMin = TextEditingController();
  final _ctrlMax = TextEditingController();

  @override
  void initState() {
    super.initState();
    _categoryId = widget.activeFilters.categoryId;
    _ordering = widget.activeFilters.ordering;
    _ctrlMin.text = widget.activeFilters.minPrice?.toStringAsFixed(0) ?? '';
    _ctrlMax.text = widget.activeFilters.maxPrice?.toStringAsFixed(0) ?? '';
  }

  @override
  void dispose() {
    _ctrlMin.dispose();
    _ctrlMax.dispose();
    super.dispose();
  }

  void _apply() {
    Navigator.pop(
      context,
      ProductFilters(
        categoryId: _categoryId,
        ordering: _ordering,
        minPrice: double.tryParse(_ctrlMin.text),
        maxPrice: double.tryParse(_ctrlMax.text),
      ),
    );
  }

  void _clear() {
    Navigator.pop(context, const ProductFilters());
  }

  @override
  Widget build(BuildContext context) {
    return DraggableScrollableSheet(
      initialChildSize: 0.75,
      minChildSize: 0.5,
      maxChildSize: 0.95,
      expand: false,
      builder: (_, scrollCtrl) => Column(
        children: [
          // Handle
          Container(
            margin: const EdgeInsets.symmetric(vertical: 12),
            width: 40,
            height: 4,
            decoration: BoxDecoration(
              color: AppColors.border,
              borderRadius: BorderRadius.circular(2),
            ),
          ),
          // Header
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 20),
            child: Row(
              children: [
                const Text('Filters',
                    style: TextStyle(fontSize: 18, fontWeight: FontWeight.w700)),
                const Spacer(),
                TextButton(
                    onPressed: _clear,
                    child: const Text('Clear',
                        style: TextStyle(color: AppColors.error))),
              ],
            ),
          ),
          const Divider(),

          // Contenido scrollable
          Expanded(
            child: ListView(
              controller: scrollCtrl,
              padding: const EdgeInsets.all(20),
              children: [
                // ── Category ────────────────────────────────
                const _SectionTitle('Category'),
                const SizedBox(height: 8),
                Wrap(
                  spacing: 8,
                  runSpacing: 8,
                  children: [
                    _FilterChip(
                      label: 'All',
                      active: _categoryId == null,
                      onTap: () => setState(() => _categoryId = null),
                    ),
                    ...widget.categories.map((cat) => _FilterChip(
                      label: cat.name,
                      active: _categoryId == cat.id,
                      onTap: () => setState(() => _categoryId = cat.id),
                    )),
                  ],
                ),
                const SizedBox(height: 20),

                // ── Price range ───────────────────────────
                const _SectionTitle('Price Range'),
                const SizedBox(height: 8),
                Row(
                  children: [
                    Expanded(
                      child: TextField(
                        controller: _ctrlMin,
                        keyboardType: TextInputType.number,
                        decoration: const InputDecoration(
                          labelText: 'Min',
                          prefixText: '\$ ',
                        ),
                      ),
                    ),
                    const SizedBox(width: 12),
                    const Text('—',
                        style: TextStyle(
                            color: AppColors.textFaint, fontSize: 18)),
                    const SizedBox(width: 12),
                    Expanded(
                      child: TextField(
                        controller: _ctrlMax,
                        keyboardType: TextInputType.number,
                        decoration: const InputDecoration(
                          labelText: 'Max',
                          prefixText: '\$ ',
                        ),
                      ),
                    ),
                  ],
                ),
                const SizedBox(height: 20),

                // ── Sort by ───────────────────────────────
                const _SectionTitle('Sort by'),
                const SizedBox(height: 8),
                ..._orderOptions.map((o) => RadioListTile<String>(
                      title: Text(o.$1),
                      value: o.$2,
                      groupValue: _ordering,
                      onChanged: (v) => setState(() => _ordering = v),
                      activeColor: AppColors.accent,
                      contentPadding: EdgeInsets.zero,
                    )),
              ],
            ),
          ),

          // Footer
          Padding(
            padding: const EdgeInsets.fromLTRB(20, 12, 20, 20),
            child: ElevatedButton(
              onPressed: _apply,
              child: const Text('Apply filters'),
            ),
          ),
        ],
      ),
    );
  }
}

class _SectionTitle extends StatelessWidget {
  final String text;
  const _SectionTitle(this.text);

  @override
  Widget build(BuildContext context) => Text(
        text.toUpperCase(),
        style: const TextStyle(
          fontSize: 12,
          fontWeight: FontWeight.w700,
          color: AppColors.textSecondary,
          letterSpacing: 0.8,
        ),
      );
}

class _FilterChip extends StatelessWidget {
  final String label;
  final bool active;
  final VoidCallback onTap;

  const _FilterChip({required this.label, required this.active, required this.onTap});

  @override
  Widget build(BuildContext context) => GestureDetector(
        onTap: onTap,
        child: AnimatedContainer(
          duration: const Duration(milliseconds: 150),
          padding: const EdgeInsets.symmetric(horizontal: 14, vertical: 8),
          decoration: BoxDecoration(
            color: active ? AppColors.accent : AppColors.surface,
            borderRadius: BorderRadius.circular(20),
            border: Border.all(
              color: active ? AppColors.accent : AppColors.border,
              width: 1.5,
            ),
          ),
          child: Text(
            label,
            style: TextStyle(
              fontSize: 13,
              fontWeight: active ? FontWeight.w700 : FontWeight.normal,
              color: active ? AppColors.onAccent : AppColors.textSecondary,
            ),
          ),
        ),
      );
}

// ── Modelos de filtros ───────────────────────────────────────

class ProductFilters {
  final int? categoryId;
  final String? ordering;
  final double? minPrice;
  final double? maxPrice;

  const ProductFilters({
    this.categoryId,
    this.ordering,
    this.minPrice,
    this.maxPrice,
  });

  int get activeCount {
    int count = 0;
    if (categoryId != null) count++;
    if (ordering != null) count++;
    if (minPrice != null) count++;
    if (maxPrice != null) count++;
    return count;
  }
}
```

---

## 4. Catálogo actualizado con búsqueda y filtros

```dart
// lib/presentation/screens/catalog/catalog_screen.dart — versión M5

import 'package:flutter/material.dart' hide SearchBar;
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../providers/catalog_provider.dart';
import '../../widgets/product_card.dart';
import 'package:flutter_shop_app/presentation/widgets/filters_sheet.dart';
import '../../widgets/search_bar.dart';

class CatalogScreen extends ConsumerStatefulWidget {
  const CatalogScreen({super.key});

  @override
  ConsumerState<CatalogScreen> createState() => _CatalogScreenState();
}

class _CatalogScreenState extends ConsumerState<CatalogScreen> {
  final _scrollController = ScrollController();

  @override
  void initState() {
    super.initState();
    _scrollController.addListener(() {
      if (_scrollController.position.pixels >=
          _scrollController.position.maxScrollExtent - 200) {
        ref.read(catalogProvider.notifier).loadMore();
      }
    });
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  Future<void> _openFilters() async {
    final state = ref.read(catalogProvider);
    final activeFilters = ProductFilters(
      categoryId: state.categoryId,
      ordering: state.ordering,
      minPrice: state.minPrice,
      maxPrice: state.maxPrice,
    );
    final result = await showFiltersSheet(
      context: context,
      activeFilters: activeFilters,
      categories: state.categories,
    );
    if (result != null && mounted) {
      ref.read(catalogProvider.notifier).setCategory(result.categoryId);
      ref.read(catalogProvider.notifier).setOrdering(result.ordering);
      ref.read(catalogProvider.notifier).setPriceRange(result.minPrice, result.maxPrice);
    }
  }

  @override
  Widget build(BuildContext context) {
    final state = ref.watch(catalogProvider);
    final numFilters = _countActiveFilters(state);

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
            ElevatedButton(
              onPressed: () => ref.read(catalogProvider.notifier).refresh(),
              child: const Text('Retry'),
            ),
          ],
        ),
      );
    }

    return RefreshIndicator(
      color: AppColors.accent,
      onRefresh: ref.read(catalogProvider.notifier).refresh,
      child: CustomScrollView(
        controller: _scrollController,
        slivers: [
          // ── Search bar + filter button ─────────────
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.fromLTRB(16, 12, 16, 8),
              child: Row(
                children: [
                  Expanded(
                    child: SearchBar(
                      initialValue: state.search,
                      onChanged: (q) => ref.read(catalogProvider.notifier).setSearch(q),
                    ),
                  ),
                  const SizedBox(width: 8),
                  Stack(
                    children: [
                      IconButton.filled(
                        style: IconButton.styleFrom(
                          backgroundColor: numFilters > 0
                              ? AppColors.accent
                              : AppColors.surface,
                          foregroundColor: numFilters > 0
                              ? AppColors.onAccent
                              : AppColors.textPrimary,
                          shape: RoundedRectangleBorder(
                            borderRadius: BorderRadius.circular(12),
                            side: const BorderSide(color: AppColors.border),
                          ),
                        ),
                        icon: const Icon(Icons.tune),
                        onPressed: _openFilters,
                      ),
                      if (numFilters > 0)
                        Positioned(
                          top: 4,
                          right: 4,
                          child: Container(
                            width: 16,
                            height: 16,
                            decoration: const BoxDecoration(
                              color: AppColors.error,
                              shape: BoxShape.circle,
                            ),
                            child: Center(
                              child: Text(
                                '$numFilters',
                                style: const TextStyle(
                                  color: Colors.white,
                                  fontSize: 10,
                                  fontWeight: FontWeight.w700,
                                ),
                              ),
                            ),
                          ),
                        ),
                    ],
                  ),
                ],
              ),
            ),
          ),

          // ── Results count ──────────────────────────────
          SliverToBoxAdapter(
            child: Padding(
              padding: const EdgeInsets.symmetric(horizontal: 16),
              child: Text(
                '${state.total} result${state.total != 1 ? 's' : ''}',
                style: const TextStyle(fontSize: 13, color: AppColors.textSecondary),
              ),
            ),
          ),

          // ── Grid ────────────────────────────────────────
          if (state.products.isEmpty && !state.isLoading)
            const SliverFillRemaining(
              child: Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Text('📦', style: TextStyle(fontSize: 52)),
                    SizedBox(height: 16),
                    Text('No results',
                        style: TextStyle(
                            color: AppColors.textPrimary,
                            fontSize: 18,
                            fontWeight: FontWeight.bold)),
                  ],
                ),
              ),
            )
          else
            SliverPadding(
              padding: const EdgeInsets.all(16),
              sliver: SliverGrid(
                gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                  crossAxisCount: 2,
                  mainAxisSpacing: 8,
                  crossAxisSpacing: 8,
                  childAspectRatio: 0.72,
                ),
                delegate: SliverChildBuilderDelegate(
                  (_, i) {
                    final p = state.products[i];
                    return ProductCard(
                      product: p,
                      onTap: () => context.push('/catalog/${p.id}'),
                    );
                  },
                  childCount: state.products.length,
                ),
              ),
            ),

          // ── Loading more spinner ────────────────────────
          if (state.isLoadingMore)
            const SliverToBoxAdapter(
              child: Padding(
                padding: EdgeInsets.all(16),
                child: Center(
                  child: CircularProgressIndicator(color: AppColors.accent),
                ),
              ),
            ),
          const SliverToBoxAdapter(child: SizedBox(height: 32)),
        ],
      ),
    );
  }

  int _countActiveFilters(CatalogState state) {
    int count = 0;
    if (state.categoryId != null) count++;
    if (state.ordering != null) count++;
    if (state.minPrice != null) count++;
    if (state.maxPrice != null) count++;
    return count;
  }
}
```

---

## 5. Pantalla de detalle — `presentation/screens/catalog/product_detail_screen.dart`

```dart
// lib/presentation/screens/catalog/product_detail_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/formatters.dart';
import '../../../core/config/app_config.dart';
import '../../../domain/model/product.dart';
import '../../providers/catalog_provider.dart';

class ProductDetailScreen extends ConsumerWidget {
  final int productId;
  const ProductDetailScreen({super.key, required this.productId});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(catalogProvider);
    final product = state.products.firstWhere(
      (p) => p.id == productId,
      orElse: () => state.products.isEmpty
          ? Product.empty()
          : state.products.first,
    );

    if (state.isLoading && state.products.isEmpty) {
      return const Scaffold(
        body: Center(child: CircularProgressIndicator(color: AppColors.accent)),
      );
    }

    if (product.id == 0) {
      return Scaffold(
        appBar: AppBar(),
        body: const Center(
          child: Text('Product not found', style: TextStyle(color: AppColors.error)),
        ),
      );
    }

    return _ProductDetailContent(product: product);
  }
}

class _ProductDetailContent extends StatefulWidget {
  final Product product;
  const _ProductDetailContent({required this.product});

  @override
  State<_ProductDetailContent> createState() => _ProductDetailContentState();
}

class _ProductDetailContentState extends State<_ProductDetailContent> {
  int _quantity = 1;

  @override
  Widget build(BuildContext context) {
    final p = widget.product;
    final outOfStock = p.stock == 0;
    final subtotal = p.price * _quantity;
    final taxAmount = subtotal * AppConfig.taxRate;
    final totalWithTax = subtotal + taxAmount;

    return Scaffold(
      appBar: AppBar(title: Text(p.name, overflow: TextOverflow.ellipsis)),
      body: SingleChildScrollView(
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            // ── Image ─────────────────────────────────────
            Stack(
              children: [
                Container(
                  height: 240,
                  width: double.infinity,
                  color: AppColors.borderLight,
                  child: const Center(
                    child: Text('📦', style: TextStyle(fontSize: 72)),
                  ),
                ),
                if (outOfStock)
                  Positioned(
                    bottom: 0,
                    left: 0,
                    right: 0,
                    child: Container(
                      color: AppColors.error.withValues(alpha: 0.85),
                      padding: const EdgeInsets.symmetric(vertical: 10),
                      child: const Text(
                        'OUT OF STOCK',
                        textAlign: TextAlign.center,
                        style: TextStyle(
                          color: Colors.white,
                          fontWeight: FontWeight.w800,
                          fontSize: 15,
                          letterSpacing: 2,
                        ),
                      ),
                    ),
                  ),
              ],
            ),

            // ── Body ─────────────────────────────────────
            Padding(
              padding: const EdgeInsets.all(20),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  // Category
                  Text(
                    (p.category?.name ?? 'No category').toUpperCase(),
                    style: const TextStyle(
                      fontSize: 11,
                      color: AppColors.accent,
                      fontWeight: FontWeight.w700,
                      letterSpacing: 0.8,
                    ),
                  ),
                  const SizedBox(height: 4),

                  // Name
                  Text(
                    p.name,
                    style: const TextStyle(
                      fontSize: 22,
                      fontWeight: FontWeight.w800,
                      color: AppColors.textPrimary,
                    ),
                  ),
                  const SizedBox(height: 8),

                  // Prices
                  Text(
                    formatPrice(p.price),
                    style: const TextStyle(
                      fontSize: 28,
                      fontWeight: FontWeight.w800,
                      color: AppColors.accent,
                    ),
                  ),
                  Text(
                    '${formatPrice(totalWithTax)} with tax (${(AppConfig.taxRate * 100).toInt()}%)',
                    style: const TextStyle(fontSize: 13, color: AppColors.textSecondary),
                  ),
                  const SizedBox(height: 8),

                  // Stock
                  Row(
                    children: [
                      Container(
                        width: 8,
                        height: 8,
                        decoration: BoxDecoration(
                          shape: BoxShape.circle,
                          color: outOfStock ? AppColors.error : AppColors.success,
                        ),
                      ),
                      const SizedBox(width: 7),
                      Text(
                        outOfStock
                            ? 'Product out of stock'
                            : '${p.stock} units in stock',
                        style: const TextStyle(
                            color: AppColors.textSecondary, fontSize: 14),
                      ),
                    ],
                  ),

                  // Description
                  if (p.description.isNotEmpty) ...[
                    const SizedBox(height: 16),
                    const Text(
                      'Description',
                      style: TextStyle(
                        fontSize: 15,
                        fontWeight: FontWeight.w700,
                        color: AppColors.textPrimary,
                      ),
                    ),
                    const SizedBox(height: 4),
                    Text(
                      p.description,
                      style: const TextStyle(
                        fontSize: 15,
                        color: AppColors.textSecondary,
                        height: 1.5,
                      ),
                    ),
                  ],

                  // Quantity selector
                  if (!outOfStock) ...[
                    const SizedBox(height: 16),
                    const Text(
                      'Quantity',
                      style: TextStyle(
                        fontSize: 15,
                        fontWeight: FontWeight.w700,
                        color: AppColors.textPrimary,
                      ),
                    ),
                    const SizedBox(height: 4),
                    Row(
                      children: [
                        _QuantityButton(
                          icon: Icons.remove,
                          onTap: _quantity > 1
                              ? () => setState(() => _quantity--)
                              : null,
                        ),
                        Padding(
                          padding: const EdgeInsets.symmetric(horizontal: 20),
                          child: Text(
                            '$_quantity',
                            style: const TextStyle(
                              fontSize: 24,
                              fontWeight: FontWeight.w800,
                              color: AppColors.textPrimary,
                            ),
                          ),
                        ),
                        _QuantityButton(
                          icon: Icons.add,
                          onTap: _quantity < p.stock
                              ? () => setState(() => _quantity++)
                              : null,
                        ),
                      ],
                    ),
                    const SizedBox(height: 16),

                    // Subtotal
                    Container(
                      padding: const EdgeInsets.symmetric(vertical: 12),
                      decoration: const BoxDecoration(
                        border: Border(top: BorderSide(color: AppColors.border)),
                      ),
                      child: Row(
                        mainAxisAlignment: MainAxisAlignment.spaceBetween,
                        children: [
                          const Text(
                            'Subtotal:',
                            style: TextStyle(
                                fontSize: 16, color: AppColors.textSecondary),
                          ),
                          Text(
                            formatPrice(totalWithTax),
                            style: const TextStyle(
                              fontSize: 20,
                              fontWeight: FontWeight.w800,
                              color: AppColors.accent,
                            ),
                          ),
                        ],
                      ),
                    ),
                  ],

                  // Add to cart button
                  const SizedBox(height: 8),
                  ElevatedButton(
                    style: outOfStock
                        ? ElevatedButton.styleFrom(
                            backgroundColor: AppColors.textFaint)
                        : null,
                    onPressed: outOfStock
                        ? null
                        : () {
                            // In M6 we'll connect to CartNotifier
                            ScaffoldMessenger.of(context).showSnackBar(
                              SnackBar(
                                content: Text(
                                    '✅ $_quantity× ${p.name} — In M6 we add to real cart'),
                                backgroundColor: AppColors.success,
                              ),
                            );
                          },
                    child: Text(
                      outOfStock
                          ? 'Out of stock'
                          : 'Add${_quantity > 1 ? ' $_quantity×' : ''} to cart',
                    ),
                  ),

                  // Meta
                  const SizedBox(height: 20),
                  Text(
                    'Updated: ${formatDateTime(p.updatedAt)}',
                    style: const TextStyle(
                        fontSize: 12, color: AppColors.textFaint),
                    textAlign: TextAlign.center,
                  ),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class _QuantityButton extends StatelessWidget {
  final IconData icon;
  final VoidCallback? onTap;

  const _QuantityButton({required this.icon, this.onTap});

  @override
  Widget build(BuildContext context) => IconButton(
        style: IconButton.styleFrom(
          backgroundColor: AppColors.surface,
          shape: RoundedRectangleBorder(
            borderRadius: BorderRadius.circular(12),
            side: const BorderSide(color: AppColors.border, width: 1.5),
          ),
        ),
        icon: Icon(icon,
            color: onTap != null
                ? AppColors.textPrimary
                : AppColors.textFaint),
        onPressed: onTap,
      );
}
```

---

## 6. Actualizar el router con la ruta de detalle

```dart
// lib/presentation/navigation/app_router.dart — añadir dentro del ShellRoute

import '../screens/catalog/product_detail_screen.dart';

// Dentro de routes del ShellRoute:
GoRoute(
  path: '/catalog',
  builder: (_, __) => const CatalogScreen(),
  routes: [
    GoRoute(
      path: ':id', // /catalog/1 → id=1
      builder: (_, state) {
        final id = int.parse(state.pathParameters['id']!);
        return ProductDetailScreen(productId: id);
      },
    ),
  ],
),
```

---

## ✅ Checkpoint — Módulo 5

| Test | Expected result |
|---|---|
| Search bar visible | With search icon |
| Type "laptop" | Waits 500 ms then filters |
| Clear search | All products return |
| Filter button (tune icon) | Opens BottomSheet |
| Select category in filters | Chip with blue color |
| Ingresar precio máximo 100 | Solo productos ≤ 100€ |
| Ordenar por precio mayor | Más caros primero |
| Badge rojo en botón filtros | Número de filtros activos |
| Limpiar en el sheet | Vuelve la lista completa |
| Tocar un producto | Navega a la pantalla de detalle |
| Precio con IVA en detalle | `121.00 €  con IVA (21%)` |
| Botones `+` y `-` de cantidad | Cambian el subtotal en tiempo real |
| Producto agotado | Botón gris, banner rojo |
| Botón "Añadir al carrito" | SnackBar verde — real en M6 |
| Flecha atrás | Vuelve al catálogo con el estado preservado |

---

## Resumen del Módulo 5

- `Buscador` usa `Timer` + cancelación para el debounce — patrón equivalente al `setTimeout/clearTimeout` de React Native.
- `DraggableScrollableSheet` dentro del `showModalBottomSheet` da el sheet con tamaño ajustable al arrastrar.
- `RadioListTile` es el widget natural para la selección de ordenamiento — una opción activa a la vez.
- `context.push('/catalogo/${p.id}')` navega al detalle sin salir del ShellRoute — la tab bar permanece visible.
- `state.pathParameters['id']` en go_router lee el segmento dinámico de la URL.
- `SliverGrid` con `childAspectRatio: 0.72` controla la altura de cada tarjeta en la grilla.
- `AnimatedContainer` en el chip de filtros da la transición suave de color al seleccionar.

---

> **Siguiente módulo →** Módulo 6: Carrito y creación de pedido —
> `CarritoProvider`, pantalla de carrito y confirmar compra en la API de Django.
