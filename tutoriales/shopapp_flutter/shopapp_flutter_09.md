# Flutter Shop App — Módulo 9
## Admin CRUD Productos — Lista, crear, editar, restock e imagen

> **Objetivo:** CRUD completo de productos con filtros por stock/estado, formulario completo con selector de categoría, toggle optimista y modal de restock con Snackbar.
> **Checkpoint final:** crear un producto, hacer restock, verificar el badge de stock en el catálogo público.

---

## 9.1 Provider de productos admin — `presentation/providers/products_admin_provider.dart`

```dart
// lib/presentation/providers/products_admin_provider.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../data/remote/api/product_remote_datasource.dart';
import '../../domain/model/category.dart';
import '../../domain/model/product.dart';

enum ProductStockFilter { all, inStock, outOfStock, active, inactive }

extension ProductStockFilterLabel on ProductStockFilter {
  String get label => switch (this) {
    ProductStockFilter.all        => 'Todos',
    ProductStockFilter.inStock    => 'Con stock',
    ProductStockFilter.outOfStock => 'Sin stock',
    ProductStockFilter.active     => 'Activos',
    ProductStockFilter.inactive   => 'Inactivos',
  };
}

class ProductsAdminState {
  final List<Product>      products;
  final bool               isLoading;
  final String?            error;
  final int                total;
  final String             search;
  final ProductStockFilter stockFilter;

  const ProductsAdminState({
    this.products    = const [],
    this.isLoading   = false,
    this.error,
    this.total       = 0,
    this.search      = '',
    this.stockFilter = ProductStockFilter.all,
  });

  List<Product> get filtered => products.where((p) {
    final matchSearch = search.isEmpty ||
        p.name.toLowerCase().contains(search.toLowerCase());
    final matchFilter = switch (stockFilter) {
      ProductStockFilter.all        => true,
      ProductStockFilter.inStock    => p.stock > 0,
      ProductStockFilter.outOfStock => p.stock == 0,
      ProductStockFilter.active     => p.isActive,
      ProductStockFilter.inactive   => !p.isActive,
    };
    return matchSearch && matchFilter;
  }).toList();

  ProductsAdminState copyWith({
    List<Product>?      products,
    bool?               isLoading,
    String?             error,
    int?                total,
    String?             search,
    ProductStockFilter? stockFilter,
  }) => ProductsAdminState(
    products:    products    ?? this.products,
    isLoading:   isLoading   ?? this.isLoading,
    error:       error,
    total:       total       ?? this.total,
    search:      search      ?? this.search,
    stockFilter: stockFilter ?? this.stockFilter,
  );
}

sealed class ProductFormState { const ProductFormState(); }
class ProductFormIdle    extends ProductFormState { const ProductFormIdle(); }
class ProductFormSaving  extends ProductFormState { const ProductFormSaving(); }
class ProductFormSuccess extends ProductFormState {
  final String message;
  const ProductFormSuccess(this.message);
}
class ProductFormError extends ProductFormState {
  final String message;
  const ProductFormError(this.message);
}

class ProductsAdminNotifier extends StateNotifier<ProductsAdminState> {
  final ProductRemoteDatasource _datasource;

  ProductsAdminNotifier(this._datasource) : super(const ProductsAdminState()) {
    load();
  }

  final _formState = StateController<ProductFormState>(const ProductFormIdle());
  ProductFormState get formState => _formState.state;

  Future<void> load() async {
    state = state.copyWith(isLoading: true, error: null);
    try {
      final result = await _datasource.getProducts(pageSize: 50);
      state = state.copyWith(
        products:  result.results,
        total:     result.count,
        isLoading: false,
      );
    } catch (e) {
      state = state.copyWith(
        isLoading: false,
        error:     e.toString().replaceAll('Exception: ', ''),
      );
    }
  }

  void setSearch(String q)           => state = state.copyWith(search: q);
  void setStockFilter(ProductStockFilter f) => state = state.copyWith(stockFilter: f);

  // Toggle optimista
  void toggleActive(int id, bool isActive) {
    state = state.copyWith(
      products: state.products.map((p) =>
        p.id == id ? p.copyWith(isActive: isActive) : p,
      ).toList(),
    );
    _datasource.updateProduct(id, {'is_active': isActive}).catchError((_) {
      state = state.copyWith(
        products: state.products.map((p) =>
          p.id == id ? p.copyWith(isActive: !isActive) : p,
        ).toList(),
      );
    });
  }

  Future<void> createProduct(Map<String, dynamic> payload) async {
    _formState.state = const ProductFormSaving();
    try {
      final created = await _datasource.createProduct(payload);
      state = state.copyWith(
        products: [created, ...state.products],
        total:    state.total + 1,
      );
      _formState.state = const ProductFormSuccess('Producto creado');
    } catch (e) {
      _formState.state = ProductFormError(e.toString().replaceAll('Exception: ', ''));
    }
  }

  Future<void> updateProduct(int id, Map<String, dynamic> payload) async {
    _formState.state = const ProductFormSaving();
    try {
      final updated = await _datasource.updateProduct(id, payload);
      state = state.copyWith(
        products: state.products.map((p) => p.id == id ? updated : p).toList(),
      );
      _formState.state = const ProductFormSuccess('Producto actualizado');
    } catch (e) {
      _formState.state = ProductFormError(e.toString().replaceAll('Exception: ', ''));
    }
  }

  // Restock — devuelve el nuevo stock
  Future<int?> restock(int id, int quantity) async {
    try {
      final result   = await _datasource.restock(id, quantity);
      final newStock = result['new_stock'] as int;
      state = state.copyWith(
        products: state.products.map((p) =>
          p.id == id ? p.copyWith(stock: newStock) : p,
        ).toList(),
      );
      return newStock;
    } catch (e) {
      return null;
    }
  }

  Future<void> deleteProduct(int id) async {
    try {
      await _datasource.deleteProduct(id);
      state = state.copyWith(
        products: state.products.where((p) => p.id != id).toList(),
        total:    state.total - 1,
      );
    } catch (e) {
      state = state.copyWith(error: e.toString().replaceAll('Exception: ', ''));
    }
  }

  void resetFormState() => _formState.state = const ProductFormIdle();
}

final productsAdminProvider =
    StateNotifierProvider<ProductsAdminNotifier, ProductsAdminState>((ref) {
  return ProductsAdminNotifier(ref.watch(productDatasourceProvider));
});
```

---

## 9.2 Formulario de producto — `presentation/widgets/product_form.dart`

```dart
// lib/presentation/widgets/product_form.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/validators.dart';
import '../../domain/model/category.dart';
import '../../domain/model/product.dart';
import '../providers/products_admin_provider.dart';

Future<void> showProductForm(
  BuildContext context,
  WidgetRef    ref, {
  Product?          initial,
  required List<Category> categories,
}) {
  return showModalBottomSheet(
    context:           context,
    isScrollControlled:true,
    backgroundColor:   AppColors.surface,
    shape: const RoundedRectangleBorder(
      borderRadius: BorderRadius.vertical(top: Radius.circular(24)),
    ),
    builder: (_) => ProviderScope(
      parent: ProviderScope.containerOf(context),
      child:  ProductFormSheet(initial: initial, categories: categories),
    ),
  );
}

class ProductFormSheet extends ConsumerStatefulWidget {
  final Product?       initial;
  final List<Category> categories;
  const ProductFormSheet({super.key, this.initial, required this.categories});

  @override
  ConsumerState<ProductFormSheet> createState() => _ProductFormSheetState();
}

class _ProductFormSheetState extends ConsumerState<ProductFormSheet> {
  final _formKey   = GlobalKey<FormState>();
  final _nameCtrl  = TextEditingController();
  final _descCtrl  = TextEditingController();
  final _priceCtrl = TextEditingController();
  final _stockCtrl = TextEditingController();
  bool     _isActive   = true;
  int?     _categoryId;

  @override
  void initState() {
    super.initState();
    if (widget.initial != null) {
      final p        = widget.initial!;
      _nameCtrl.text  = p.name;
      _descCtrl.text  = p.description;
      _priceCtrl.text = p.price.toStringAsFixed(2);
      _stockCtrl.text = p.stock.toString();
      _isActive       = p.isActive;
      _categoryId     = p.category?.id;
    }
  }

  @override
  void dispose() {
    _nameCtrl.dispose();
    _descCtrl.dispose();
    _priceCtrl.dispose();
    _stockCtrl.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;
    final payload = {
      'name':        _nameCtrl.text.trim(),
      'description': _descCtrl.text.trim(),
      'price':       double.parse(_priceCtrl.text),
      'stock':       int.parse(_stockCtrl.text),
      'is_active':   _isActive,
      'category_id': _categoryId,
    };
    if (widget.initial != null) {
      await ref.read(productsAdminProvider.notifier)
          .updateProduct(widget.initial!.id, payload);
    } else {
      await ref.read(productsAdminProvider.notifier).createProduct(payload);
    }
  }

  @override
  Widget build(BuildContext context) {
    final notifier = ref.read(productsAdminProvider.notifier);
    final formSt   = notifier.formState;
    final isSaving = formSt is ProductFormSaving;
    final isEdit   = widget.initial != null;

    if (formSt is ProductFormSuccess) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        if (mounted) Navigator.pop(context);
      });
    }

    final activeCategories = widget.categories.where((c) => c.isActive).toList();

    return Padding(
      padding: EdgeInsets.only(bottom: MediaQuery.of(context).viewInsets.bottom),
      child:   SingleChildScrollView(
        padding: const EdgeInsets.fromLTRB(24, 8, 24, 32),
        child:   Column(
          mainAxisSize:        MainAxisSize.min,
          crossAxisAlignment:  CrossAxisAlignment.start,
          children: [
            // Drag handle
            Center(
              child: Container(
                width: 40, height: 4,
                margin:     const EdgeInsets.symmetric(vertical: 12),
                decoration: BoxDecoration(
                  color: AppColors.border, borderRadius: BorderRadius.circular(2),
                ),
              ),
            ),

            Text(
              isEdit ? 'Editar: ${widget.initial!.name}' : 'Nuevo producto',
              style: const TextStyle(
                color: AppColors.textPrimary, fontSize: 20, fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 20),

            if (formSt is ProductFormError) ...[
              Container(
                width: double.infinity,
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(
                  color: AppColors.error.withValues(alpha: 0.1),
                  borderRadius: BorderRadius.circular(10),
                ),
                child: Text(formSt.message,
                    style: const TextStyle(color: AppColors.error, fontSize: 13)),
              ),
              const SizedBox(height: 14),
            ],

            Form(
              key: _formKey,
              child: Column(
                children: [
                  // Nombre
                  TextFormField(
                    controller: _nameCtrl,
                    enabled:    !isSaving,
                    decoration: const InputDecoration(labelText: 'Nombre *'),
                    style:      const TextStyle(color: AppColors.textPrimary),
                    validator:  (v) => validateRequired(v, 'Nombre'),
                  ),
                  const SizedBox(height: 12),

                  // Descripción
                  TextFormField(
                    controller: _descCtrl,
                    enabled:    !isSaving,
                    maxLines:   3,
                    decoration: const InputDecoration(
                      labelText: 'Descripción',
                      alignLabelWithHint: true,
                    ),
                    style: const TextStyle(color: AppColors.textPrimary),
                  ),
                  const SizedBox(height: 12),

                  // Precio y Stock en fila
                  Row(
                    children: [
                      Expanded(
                        child: TextFormField(
                          controller:  _priceCtrl,
                          enabled:     !isSaving,
                          keyboardType:const TextInputType.numberWithOptions(decimal: true),
                          decoration:  const InputDecoration(
                            labelText: 'Precio $ *',
                            prefixText:'\$ ',
                          ),
                          style:       const TextStyle(color: AppColors.textPrimary),
                          validator:   (v) => validatePositiveNumber(v, 'Precio'),
                        ),
                      ),
                      const SizedBox(width: 12),
                      Expanded(
                        child: TextFormField(
                          controller:  _stockCtrl,
                          enabled:     !isSaving,
                          keyboardType:TextInputType.number,
                          decoration:  const InputDecoration(labelText: 'Stock *'),
                          style:       const TextStyle(color: AppColors.textPrimary),
                          validator:   (v) => validateNonNegativeInt(v, 'Stock'),
                        ),
                      ),
                    ],
                  ),
                  const SizedBox(height: 12),

                  // Selector de categoría
                  DropdownButtonFormField<int>(
                    value:       _categoryId,
                    decoration:  const InputDecoration(labelText: 'Categoría *'),
                    dropdownColor: AppColors.surface2,
                    style:       const TextStyle(color: AppColors.textPrimary),
                    items: [
                      const DropdownMenuItem(
                        value: null,
                        child: Text('— Seleccionar —',
                            style: TextStyle(color: AppColors.textFaint)),
                      ),
                      ...activeCategories.map((c) => DropdownMenuItem(
                        value: c.id,
                        child: Text(c.name),
                      )),
                    ],
                    onChanged: isSaving ? null : (v) => setState(() => _categoryId = v),
                    validator: (v) => v == null ? 'Selecciona una categoría' : null,
                  ),
                  const SizedBox(height: 12),

                  // Toggle activo
                  Container(
                    padding:    const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
                    decoration: BoxDecoration(
                      color:        AppColors.surface2,
                      borderRadius: BorderRadius.circular(12),
                    ),
                    child: Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        const Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            Text('Producto activo',
                                style: TextStyle(color: AppColors.textPrimary, fontWeight: FontWeight.w600)),
                            Text('Visible en el catálogo',
                                style: TextStyle(color: AppColors.textSecondary, fontSize: 12)),
                          ],
                        ),
                        Switch(
                          value:       _isActive,
                          onChanged:   isSaving ? null : (v) => setState(() => _isActive = v),
                          activeColor: AppColors.accent,
                        ),
                      ],
                    ),
                  ),
                  const SizedBox(height: 20),

                  Row(
                    children: [
                      Expanded(
                        child: OutlinedButton(
                          onPressed: isSaving ? null : () => Navigator.pop(context),
                          child:     const Text('Cancelar'),
                        ),
                      ),
                      const SizedBox(width: 12),
                      Expanded(
                        child: ElevatedButton(
                          onPressed: isSaving ? null : _submit,
                          child: isSaving
                              ? const SizedBox(
                                  width: 18, height: 18,
                                  child: CircularProgressIndicator(
                                    strokeWidth: 2.5, color: AppColors.onAccent,
                                  ),
                                )
                              : Text(isEdit ? 'Guardar cambios' : 'Crear producto'),
                        ),
                      ),
                    ],
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
```

---

## 9.3 Dialog de restock — `presentation/widgets/restock_dialog.dart`

```dart
// lib/presentation/widgets/restock_dialog.dart

import 'package:flutter/material.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/validators.dart';
import '../../domain/model/product.dart';

Future<int?> showRestockDialog(BuildContext context, Product product) {
  return showDialog<int>(
    context: context,
    builder: (_) => _RestockDialog(product: product),
  );
}

class _RestockDialog extends StatefulWidget {
  final Product product;
  const _RestockDialog({required this.product});

  @override
  State<_RestockDialog> createState() => _RestockDialogState();
}

class _RestockDialogState extends State<_RestockDialog> {
  final _formKey = GlobalKey<FormState>();
  final _qtyCtrl = TextEditingController();

  @override
  void dispose() {
    _qtyCtrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final qty    = int.tryParse(_qtyCtrl.text);
    final newQty = qty != null ? widget.product.stock + qty : null;

    return AlertDialog(
      backgroundColor: AppColors.surface,
      shape:           RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      title:           Column(
        crossAxisAlignment: CrossAxisAlignment.start,
        children: [
          Text('Restock: ${widget.product.name}',
              style: const TextStyle(color: AppColors.textPrimary, fontSize: 16)),
          const SizedBox(height: 4),
          Text(
            'Stock actual: ${widget.product.stock} unidades',
            style: TextStyle(
              color:    widget.product.stock == 0 ? AppColors.error : AppColors.textSecondary,
              fontSize: 13,
            ),
          ),
        ],
      ),
      content: Form(
        key: _formKey,
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextFormField(
              controller:   _qtyCtrl,
              keyboardType: TextInputType.number,
              autofocus:    true,
              decoration:   const InputDecoration(labelText: 'Cantidad a añadir *'),
              style:        const TextStyle(color: AppColors.textPrimary),
              validator:    (v) => validatePositiveNumber(v, 'Cantidad'),
              onChanged:    (_) => setState(() {}),
            ),
            if (newQty != null) ...[
              const SizedBox(height: 8),
              Text(
                'Nuevo stock: $newQty unidades',
                style: const TextStyle(color: AppColors.textSecondary, fontSize: 12),
              ),
            ],
          ],
        ),
      ),
      actions: [
        TextButton(
          onPressed: () => Navigator.pop(context),
          child:     const Text('Cancelar'),
        ),
        ElevatedButton(
          onPressed: () {
            if (_formKey.currentState!.validate()) {
              Navigator.pop(context, int.parse(_qtyCtrl.text));
            }
          },
          child: const Text('Añadir stock'),
        ),
      ],
    );
  }
}
```

---

## 9.4 Pantalla de productos admin — `presentation/screens/admin/products_admin_screen.dart`

```dart
// lib/presentation/screens/admin/products_admin_screen.dart

import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/formatters.dart';
import '../../data/remote/api/category_remote_datasource.dart';
import '../../domain/model/category.dart';
import '../../domain/model/product.dart';
import '../providers/products_admin_provider.dart';
import '../widgets/product_form.dart';
import '../widgets/restock_dialog.dart';

class ProductsAdminScreen extends ConsumerStatefulWidget {
  const ProductsAdminScreen({super.key});

  @override
  ConsumerState<ProductsAdminScreen> createState() => _ProductsAdminScreenState();
}

class _ProductsAdminScreenState extends ConsumerState<ProductsAdminScreen> {
  List<Category> _categories = [];

  @override
  void initState() {
    super.initState();
    ref.read(categoryRepositoryProvider).getCategories().then((cats) {
      if (mounted) setState(() => _categories = cats);
    });
  }

  @override
  Widget build(BuildContext context) {
    final state    = ref.watch(productsAdminProvider);
    final filtered = state.filtered;

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
                      const Text('Productos',
                          style: TextStyle(
                            color: AppColors.textPrimary, fontSize: 22, fontWeight: FontWeight.bold,
                          )),
                      Text('${state.total} productos',
                          style: const TextStyle(color: AppColors.textSecondary, fontSize: 13)),
                    ],
                  ),
                  Row(
                    children: [
                      IconButton(
                        onPressed: () => ref.read(productsAdminProvider.notifier).load(),
                        icon:      const Icon(Icons.refresh_rounded, color: AppColors.textSecondary),
                      ),
                      ElevatedButton.icon(
                        onPressed: () => showProductForm(context, ref, categories: _categories),
                        icon:      const Icon(Icons.add, size: 18),
                        label:     const Text('Nuevo'),
                        style:     ElevatedButton.styleFrom(
                          minimumSize: const Size(0, 40),
                          padding:     const EdgeInsets.symmetric(horizontal: 16),
                        ),
                      ),
                    ],
                  ),
                ],
              ),
              const SizedBox(height: 12),

              // Búsqueda
              TextField(
                onChanged:  ref.read(productsAdminProvider.notifier).setSearch,
                decoration: const InputDecoration(
                  hintText:   'Buscar producto...',
                  prefixIcon: Icon(Icons.search_rounded, color: AppColors.textSecondary),
                  contentPadding: EdgeInsets.symmetric(vertical: 10),
                ),
                style: const TextStyle(color: AppColors.textPrimary),
              ),
              const SizedBox(height: 10),

              // Chips de filtro
              SizedBox(
                height: 34,
                child:  ListView(
                  scrollDirection: Axis.horizontal,
                  children: ProductStockFilter.values.map((f) => Padding(
                    padding: const EdgeInsets.only(right: 8),
                    child:   ChoiceChip(
                      label:     Text(f.label),
                      selected:  state.stockFilter == f,
                      onSelected:(_) =>
                          ref.read(productsAdminProvider.notifier).setStockFilter(f),
                    ),
                  )).toList(),
                ),
              ),
              const SizedBox(height: 12),
            ],
          ),
        ),

        // ── Lista ─────────────────────────────────────────────
        Expanded(
          child: Builder(builder: (_) {
            if (state.isLoading) {
              return const Center(
                child: CircularProgressIndicator(color: AppColors.accent),
              );
            }
            if (state.error != null) {
              return Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Text(state.error!, style: const TextStyle(color: AppColors.error)),
                    const SizedBox(height: 12),
                    ElevatedButton(
                      onPressed: () => ref.read(productsAdminProvider.notifier).load(),
                      child:     const Text('Reintentar'),
                    ),
                  ],
                ),
              );
            }
            if (filtered.isEmpty) {
              return const Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    Text('📦', style: TextStyle(fontSize: 48)),
                    SizedBox(height: 12),
                    Text('Sin productos',
                        style: TextStyle(
                          color: AppColors.textPrimary, fontSize: 18, fontWeight: FontWeight.bold,
                        )),
                  ],
                ),
              );
            }

            return ListView.separated(
              padding:         const EdgeInsets.all(16),
              itemCount:       filtered.length,
              separatorBuilder:(_, __) => const SizedBox(height: 10),
              itemBuilder: (_, i) => _ProductAdminCard(
                product:    filtered[i],
                onToggle:   () => ref.read(productsAdminProvider.notifier)
                    .toggleActive(filtered[i].id, !filtered[i].isActive),
                onEdit:     () => showProductForm(
                  context, ref, initial: filtered[i], categories: _categories,
                ),
                onRestock:  () async {
                  final qty = await showRestockDialog(context, filtered[i]);
                  if (qty != null && context.mounted) {
                    final newStock = await ref
                        .read(productsAdminProvider.notifier)
                        .restock(filtered[i].id, qty);
                    if (context.mounted) {
                      ScaffoldMessenger.of(context).showSnackBar(SnackBar(
                        content: Text(
                          newStock != null
                              ? '✅ Stock actualizado: $newStock unidades'
                              : '❌ Error al actualizar el stock',
                        ),
                        backgroundColor:
                            newStock != null ? AppColors.success : AppColors.error,
                      ));
                    }
                  }
                },
                onDelete:   () => _confirmDelete(context, ref, filtered[i]),
              ),
            );
          }),
        ),
      ],
    );
  }

  void _confirmDelete(BuildContext context, WidgetRef ref, Product product) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        backgroundColor: AppColors.surface,
        shape:           RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
        title:  const Text('¿Eliminar producto?',
            style: TextStyle(color: AppColors.textPrimary)),
        content:Text('"${product.name}" se eliminará permanentemente.',
            style: const TextStyle(color: AppColors.textSecondary)),
        actions:[
          TextButton(
            onPressed: () => Navigator.pop(context),
            child:     const Text('Cancelar'),
          ),
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              ref.read(productsAdminProvider.notifier).deleteProduct(product.id);
            },
            child: const Text('Eliminar',
                style: TextStyle(color: AppColors.error, fontWeight: FontWeight.bold)),
          ),
        ],
      ),
    );
  }
}

// ── ProductAdminCard ──────────────────────────────────────────

class _ProductAdminCard extends StatelessWidget {
  final Product      product;
  final VoidCallback onToggle;
  final VoidCallback onEdit;
  final VoidCallback onRestock;
  final VoidCallback onDelete;

  const _ProductAdminCard({
    required this.product,
    required this.onToggle,
    required this.onEdit,
    required this.onRestock,
    required this.onDelete,
  });

  Color _stockColor() {
    if (product.stock == 0) return AppColors.error;
    if (product.stock < 5)  return AppColors.warning;
    return AppColors.success;
  }

  @override
  Widget build(BuildContext context) => Opacity(
    opacity: product.isActive ? 1.0 : 0.55,
    child:   Container(
      padding:    const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color:        AppColors.surface,
        borderRadius: BorderRadius.circular(14),
        border:       Border.all(color: AppColors.border),
      ),
      child: Row(
        children: [
          // Imagen
          ClipRRect(
            borderRadius: BorderRadius.circular(10),
            child: SizedBox(
              width: 54, height: 54,
              child: product.imageUrl != null
                  ? CachedNetworkImage(
                      imageUrl: product.imageUrl!,
                      fit:      BoxFit.cover,
                      errorWidget: (_, __, ___) => Container(
                        color: AppColors.surface2,
                        child: const Center(child: Text('📦')),
                      ),
                    )
                  : Container(
                      color: AppColors.surface2,
                      child: const Center(
                        child: Text('📦', style: TextStyle(fontSize: 22)),
                      ),
                    ),
            ),
          ),
          const SizedBox(width: 12),

          // Info
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  product.name,
                  style: const TextStyle(
                    color: AppColors.textPrimary, fontWeight: FontWeight.w600,
                  ),
                  maxLines: 1, overflow: TextOverflow.ellipsis,
                ),
                if (product.category != null)
                  Text(product.category!.name,
                      style: const TextStyle(color: AppColors.textSecondary, fontSize: 12)),
                Row(
                  children: [
                    Text(
                      formatPrice(product.price),
                      style: const TextStyle(
                        color: AppColors.accent, fontWeight: FontWeight.bold, fontSize: 14,
                      ),
                    ),
                    const SizedBox(width: 8),
                    Container(
                      padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 2),
                      decoration: BoxDecoration(
                        color:        _stockColor().withValues(alpha: 0.12),
                        borderRadius: BorderRadius.circular(6),
                      ),
                      child: Text(
                        product.stock == 0 ? 'Agotado' : '${product.stock} uds.',
                        style: TextStyle(
                          color:      _stockColor(),
                          fontSize:   10,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                    ),
                  ],
                ),
              ],
            ),
          ),

          // Acciones
          Column(
            children: [
              Switch(
                value:       product.isActive,
                onChanged:   (_) => onToggle(),
                activeColor: AppColors.accent,
              ),
              Row(
                children: [
                  _ActionIcon(icon: Icons.inventory_2_outlined, color: AppColors.accent,    onTap: onRestock),
                  _ActionIcon(icon: Icons.edit_outlined,         color: AppColors.textSecondary, onTap: onEdit),
                  _ActionIcon(icon: Icons.delete_outline,        color: AppColors.error,    onTap: onDelete),
                ],
              ),
            ],
          ),
        ],
      ),
    ),
  );
}

class _ActionIcon extends StatelessWidget {
  final IconData     icon;
  final Color        color;
  final VoidCallback onTap;
  const _ActionIcon({required this.icon, required this.color, required this.onTap});

  @override
  Widget build(BuildContext context) => GestureDetector(
    onTap: onTap,
    child: Padding(
      padding: const EdgeInsets.all(4),
      child:   Icon(icon, color: color, size: 20),
    ),
  );
}
```

---

## 9.5 Actualizar el router

```dart
// lib/presentation/navigation/app_router.dart — reemplazar placeholder de productos

import '../../presentation/screens/admin/products_admin_screen.dart';

GoRoute(
  path:    '/admin/products',
  builder: (_, state) => AdminShell(
    title:        'Productos',
    currentRoute: state.matchedLocation,
    child:        const ProductsAdminScreen(),
  ),
),
```

---

## ✅ Checkpoint Módulo 9

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Drawer → "Productos" | Lista con imagen, precio y badge de stock |
| 2 | Filtro "Sin stock" | Solo productos agotados |
| 3 | Buscar por nombre | Filtrado local instantáneo |
| 4 | Toggle activo | Cambia al instante (optimista) |
| 5 | Ícono 📦 Restock | `AlertDialog` con stock actual |
| 6 | Restock con 50 | Snackbar "✅ Stock actualizado: N unidades" |
| 7 | Botón "+ Nuevo" | BottomSheet con formulario |
| 8 | Crear sin categoría | Error de validación en el campo |
| 9 | Precio negativo | "Precio inválido" |
| 10 | Crear producto válido | Aparece en la lista |
| 11 | Editar precio | Precio actualizado en la lista |
| 12 | Verificar en `/catalog` | Nuevo producto con badge de stock correcto |

---

## Resumen

| Elemento | Estado |
|---|---|
| `ProductsAdminNotifier` con toggle optimista y restock | ✅ |
| `filtered` getter que combina búsqueda + filtro de stock | ✅ |
| `ProductStockFilter` enum con extension `label` | ✅ |
| `ProductFormSheet` con `DropdownButtonFormField` | ✅ |
| `showRestockDialog` con preview del nuevo stock | ✅ |
| `ProductsAdminScreen` con Snackbar tras restock | ✅ |
| `_ProductAdminCard` con imagen, badge, switch y 3 acciones | ✅ |
| Ruta `admin/products` en el router | ✅ |

**Siguiente módulo →** M10: Admin CRUD Pedidos — lista con filtros, cambio de estado inline y detalle
