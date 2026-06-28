# Flutter Shop App — Módulo 8
## Admin CRUD Categorías — Lista, crear, editar y toggle optimista

> **Objetivo:** CRUD completo de categorías desde el panel admin con búsqueda local, toggle de estado optimista con rollback, formulario en `showModalBottomSheet` y slug auto-generado.
> **Checkpoint final:** crear, editar y activar/desactivar categorías — verificar los cambios en el catálogo público de la app.

---

## 8.1 Provider de categorías admin — `presentation/providers/categories_admin_provider.dart`

```dart
// lib/presentation/providers/categories_admin_provider.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../data/remote/api/category_remote_datasource.dart';
import '../../domain/model/category.dart';

class CategoriesAdminState {
  final List<Category> categories;
  final bool           isLoading;
  final String?        error;
  final String         search;

  const CategoriesAdminState({
    this.categories = const [],
    this.isLoading  = false,
    this.error,
    this.search     = '',
  });

  List<Category> get filtered => search.isEmpty
      ? categories
      : categories.where((c) =>
          c.name.toLowerCase().contains(search.toLowerCase())).toList();

  CategoriesAdminState copyWith({
    List<Category>? categories,
    bool?           isLoading,
    String?         error,
    String?         search,
  }) => CategoriesAdminState(
    categories: categories ?? this.categories,
    isLoading:  isLoading  ?? this.isLoading,
    error:      error,
    search:     search     ?? this.search,
  );
}

sealed class CategoryFormState {
  const CategoryFormState();
}
class CategoryFormIdle    extends CategoryFormState { const CategoryFormIdle(); }
class CategoryFormSaving  extends CategoryFormState { const CategoryFormSaving(); }
class CategoryFormSuccess extends CategoryFormState {
  final String message;
  const CategoryFormSuccess(this.message);
}
class CategoryFormError extends CategoryFormState {
  final String message;
  const CategoryFormError(this.message);
}

class CategoriesAdminNotifier extends StateNotifier<CategoriesAdminState> {
  final CategoryRemoteDatasource _datasource;

  CategoriesAdminNotifier(this._datasource) : super(const CategoriesAdminState()) {
    load();
  }

  final _formState = StateController<CategoryFormState>(const CategoryFormIdle());
  CategoryFormState get formState => _formState.state;

  Future<void> load() async {
    state = state.copyWith(isLoading: true, error: null);
    try {
      final cats = await _datasource.getCategories();
      state = state.copyWith(categories: cats, isLoading: false);
    } catch (e) {
      state = state.copyWith(
        isLoading: false,
        error:     e.toString().replaceAll('Exception: ', ''),
      );
    }
  }

  void setSearch(String q) => state = state.copyWith(search: q);

  // Toggle optimista
  void toggleActive(int id, bool isActive) {
    state = state.copyWith(
      categories: state.categories.map((c) =>
        c.id == id ? c.copyWith(isActive: isActive) : c,
      ).toList(),
    );
    _datasource.updateCategory(id, {'is_active': isActive}).catchError((_) {
      // Revertir
      state = state.copyWith(
        categories: state.categories.map((c) =>
          c.id == id ? c.copyWith(isActive: !isActive) : c,
        ).toList(),
      );
    });
  }

  Future<void> createCategory(Map<String, dynamic> payload) async {
    _formState.state = const CategoryFormSaving();
    try {
      final created = await _datasource.createCategory(payload);
      state = state.copyWith(categories: [created, ...state.categories]);
      _formState.state = const CategoryFormSuccess('Categoría creada');
    } catch (e) {
      _formState.state = CategoryFormError(e.toString().replaceAll('Exception: ', ''));
    }
  }

  Future<void> updateCategory(int id, Map<String, dynamic> payload) async {
    _formState.state = const CategoryFormSaving();
    try {
      final updated = await _datasource.updateCategory(id, payload);
      state = state.copyWith(
        categories: state.categories.map((c) => c.id == id ? updated : c).toList(),
      );
      _formState.state = const CategoryFormSuccess('Categoría actualizada');
    } catch (e) {
      _formState.state = CategoryFormError(e.toString().replaceAll('Exception: ', ''));
    }
  }

  Future<void> deleteCategory(int id) async {
    try {
      await _datasource.deleteCategory(id);
      state = state.copyWith(
        categories: state.categories.where((c) => c.id != id).toList(),
      );
    } catch (e) {
      state = state.copyWith(error: e.toString().replaceAll('Exception: ', ''));
    }
  }

  void resetFormState() => _formState.state = const CategoryFormIdle();
}

final categoriesAdminProvider =
    StateNotifierProvider<CategoriesAdminNotifier, CategoriesAdminState>((ref) {
  return CategoriesAdminNotifier(ref.watch(categoryDatasourceProvider));
});
```

---

## 8.2 Formulario de categoría — `presentation/widgets/category_form.dart`

```dart
// lib/presentation/widgets/category_form.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/validators.dart';
import '../../domain/model/category.dart';
import '../providers/categories_admin_provider.dart';

// ── Generador de slug ─────────────────────────────────────────
String toSlug(String input) => input
    .toLowerCase()
    .replaceAll(RegExp(r'[áàäâ]'), 'a')
    .replaceAll(RegExp(r'[éèëê]'), 'e')
    .replaceAll(RegExp(r'[íìïî]'), 'i')
    .replaceAll(RegExp(r'[óòöôõ]'), 'o')
    .replaceAll(RegExp(r'[úùüû]'), 'u')
    .replaceAll('ñ', 'n')
    .replaceAll(RegExp(r'[^a-z0-9\s-]'), '')
    .trim()
    .replaceAll(RegExp(r'\s+'), '-');

Future<void> showCategoryForm(
  BuildContext context,
  WidgetRef    ref, {
  Category?    initial,
}) {
  return showModalBottomSheet(
    context:           context,
    isScrollControlled:true,
    backgroundColor:   AppColors.surface,
    shape: const RoundedRectangleBorder(
      borderRadius: BorderRadius.vertical(top: Radius.circular(24)),
    ),
    builder: (_) => ProviderScope(
      parent:       ProviderScope.containerOf(context),
      child:        CategoryFormSheet(initial: initial),
    ),
  );
}

class CategoryFormSheet extends ConsumerStatefulWidget {
  final Category? initial;
  const CategoryFormSheet({super.key, this.initial});

  @override
  ConsumerState<CategoryFormSheet> createState() => _CategoryFormSheetState();
}

class _CategoryFormSheetState extends ConsumerState<CategoryFormSheet> {
  final _formKey    = GlobalKey<FormState>();
  final _nameCtrl   = TextEditingController();
  final _slugCtrl   = TextEditingController();
  final _descCtrl   = TextEditingController();
  bool  _isActive   = true;
  bool  _slugEdited = false;

  @override
  void initState() {
    super.initState();
    if (widget.initial != null) {
      final c       = widget.initial!;
      _nameCtrl.text = c.name;
      _slugCtrl.text = c.slug;
      _descCtrl.text = c.description;
      _isActive      = c.isActive;
      _slugEdited    = true;
    }
    _nameCtrl.addListener(() {
      if (!_slugEdited) {
        setState(() => _slugCtrl.text = toSlug(_nameCtrl.text));
      }
    });
  }

  @override
  void dispose() {
    _nameCtrl.dispose();
    _slugCtrl.dispose();
    _descCtrl.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;
    final payload = {
      'name':        _nameCtrl.text.trim(),
      'slug':        _slugCtrl.text.trim(),
      'description': _descCtrl.text.trim(),
      'is_active':   _isActive,
    };
    if (widget.initial != null) {
      await ref.read(categoriesAdminProvider.notifier)
          .updateCategory(widget.initial!.id, payload);
    } else {
      await ref.read(categoriesAdminProvider.notifier).createCategory(payload);
    }
  }

  @override
  Widget build(BuildContext context) {
    final notifier = ref.read(categoriesAdminProvider.notifier);
    final formSt   = notifier.formState;
    final isSaving = formSt is CategoryFormSaving;
    final isEdit   = widget.initial != null;

    // Cerrar si guardó con éxito
    if (formSt is CategoryFormSuccess) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        if (mounted) Navigator.pop(context);
      });
    }

    return Padding(
      padding: EdgeInsets.only(
        bottom: MediaQuery.of(context).viewInsets.bottom,
      ),
      child: SingleChildScrollView(
        padding: const EdgeInsets.fromLTRB(24, 8, 24, 32),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          mainAxisSize: MainAxisSize.min,
          children: [
            // Drag handle
            Center(
              child: Container(
                width:  40, height: 4,
                margin: const EdgeInsets.symmetric(vertical: 12),
                decoration: BoxDecoration(
                  color:        AppColors.border,
                  borderRadius: BorderRadius.circular(2),
                ),
              ),
            ),

            Text(
              isEdit ? 'Editar: ${widget.initial!.name}' : 'Nueva categoría',
              style: const TextStyle(
                color: AppColors.textPrimary, fontSize: 20, fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 20),

            // Error del formulario
            if (formSt is CategoryFormError) ...[
              Container(
                width:   double.infinity,
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(
                  color:        AppColors.error.withValues(alpha: 0.1),
                  borderRadius: BorderRadius.circular(10),
                ),
                child: Text(
                  formSt.message,
                  style: const TextStyle(color: AppColors.error, fontSize: 13),
                ),
              ),
              const SizedBox(height: 14),
            ],

            Form(
              key: _formKey,
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  // Nombre
                  TextFormField(
                    controller:  _nameCtrl,
                    enabled:     !isSaving,
                    decoration:  const InputDecoration(labelText: 'Nombre *'),
                    style:       const TextStyle(color: AppColors.textPrimary),
                    validator:   (v) => validateRequired(v, 'Nombre'),
                  ),
                  const SizedBox(height: 14),

                  // Slug
                  TextFormField(
                    controller:  _slugCtrl,
                    enabled:     !isSaving,
                    decoration:  InputDecoration(
                      labelText: 'Slug (URL) *',
                      helperText:'URL: /catalog?category=${_slugCtrl.text}',
                    ),
                    style:       const TextStyle(
                      color: AppColors.textPrimary, fontFamily: 'monospace',
                    ),
                    onChanged:   (_) => setState(() => _slugEdited = true),
                    validator:   (v) {
                      if (v == null || v.trim().isEmpty) return 'Slug es obligatorio';
                      if (!RegExp(r'^[a-z0-9-]+$').hasMatch(v.trim())) {
                        return 'Solo minúsculas, números y guiones';
                      }
                      return null;
                    },
                  ),
                  const SizedBox(height: 14),

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
                  const SizedBox(height: 14),

                  // Toggle activa
                  Container(
                    padding:    const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
                    decoration: BoxDecoration(
                      color:        AppColors.surface2,
                      borderRadius: BorderRadius.circular(12),
                    ),
                    child: Row(
                      mainAxisAlignment: MainAxisAlignment.spaceBetween,
                      children: [
                        Column(
                          crossAxisAlignment: CrossAxisAlignment.start,
                          children: [
                            const Text('Categoría activa',
                                style: TextStyle(
                                  color:      AppColors.textPrimary,
                                  fontWeight: FontWeight.w600,
                                )),
                            const Text('Visible en el catálogo público',
                                style: TextStyle(
                                  color: AppColors.textSecondary, fontSize: 12,
                                )),
                          ],
                        ),
                        Switch(
                          value:          _isActive,
                          onChanged:      isSaving ? null : (v) => setState(() => _isActive = v),
                          activeColor:    AppColors.accent,
                          trackColor:     WidgetStateProperty.resolveWith((s) =>
                            s.contains(WidgetState.selected)
                              ? AppColors.accent.withValues(alpha: 0.4)
                              : AppColors.border,
                          ),
                        ),
                      ],
                    ),
                  ),
                  const SizedBox(height: 20),

                  // Botones
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
                          child:     isSaving
                              ? const SizedBox(
                                  width: 18, height: 18,
                                  child: CircularProgressIndicator(
                                    strokeWidth: 2.5, color: AppColors.onAccent,
                                  ),
                                )
                              : Text(isEdit ? 'Guardar cambios' : 'Crear categoría'),
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

## 8.3 Pantalla de categorías admin — `presentation/screens/admin/categories_admin_screen.dart`

```dart
// lib/presentation/screens/admin/categories_admin_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../theme/app_colors.dart';
import '../../domain/model/category.dart';
import '../providers/categories_admin_provider.dart';
import '../widgets/category_form.dart';

class CategoriesAdminScreen extends ConsumerWidget {
  const CategoriesAdminScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state    = ref.watch(categoriesAdminProvider);
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
                      const Text('Categorías',
                          style: TextStyle(
                            color: AppColors.textPrimary,
                            fontSize: 22, fontWeight: FontWeight.bold,
                          )),
                      Text(
                        '${state.categories.length} categorías',
                        style: const TextStyle(color: AppColors.textSecondary, fontSize: 13),
                      ),
                    ],
                  ),
                  ElevatedButton.icon(
                    onPressed: () => showCategoryForm(context, ref),
                    icon:      const Icon(Icons.add, size: 18),
                    label:     const Text('Nueva'),
                    style:     ElevatedButton.styleFrom(
                      minimumSize:   const Size(0, 40),
                      padding:       const EdgeInsets.symmetric(horizontal: 16, vertical: 0),
                    ),
                  ),
                ],
              ),
              const SizedBox(height: 12),
              TextField(
                onChanged:  ref.read(categoriesAdminProvider.notifier).setSearch,
                decoration: const InputDecoration(
                  hintText:   'Buscar categoría...',
                  prefixIcon: Icon(Icons.search_rounded, color: AppColors.textSecondary),
                  contentPadding: EdgeInsets.symmetric(vertical: 10),
                ),
                style: const TextStyle(color: AppColors.textPrimary),
              ),
              const SizedBox(height: 12),
            ],
          ),
        ),

        // ── Contenido ──────────────────────────────────────
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
                    Text(state.error!,
                        style: const TextStyle(color: AppColors.error)),
                    const SizedBox(height: 12),
                    ElevatedButton(
                      onPressed: () =>
                          ref.read(categoriesAdminProvider.notifier).load(),
                      child: const Text('Reintentar'),
                    ),
                  ],
                ),
              );
            }
            if (filtered.isEmpty) {
              return Center(
                child: Column(
                  mainAxisAlignment: MainAxisAlignment.center,
                  children: [
                    const Text('🏷️', style: TextStyle(fontSize: 48)),
                    const SizedBox(height: 12),
                    Text(
                      state.search.isEmpty ? 'Sin categorías' : 'Sin resultados',
                      style: const TextStyle(
                        color: AppColors.textPrimary, fontSize: 18, fontWeight: FontWeight.bold,
                      ),
                    ),
                  ],
                ),
              );
            }

            return ListView.separated(
              padding:         const EdgeInsets.all(16),
              itemCount:       filtered.length,
              separatorBuilder:(_, __) => const SizedBox(height: 10),
              itemBuilder: (_, i) => _CategoryCard(
                category: filtered[i],
                onToggle: () => ref.read(categoriesAdminProvider.notifier)
                    .toggleActive(filtered[i].id, !filtered[i].isActive),
                onEdit:   () => showCategoryForm(context, ref, initial: filtered[i]),
                onDelete: () => _confirmDelete(context, ref, filtered[i]),
              ),
            );
          }),
        ),
      ],
    );
  }

  void _confirmDelete(BuildContext context, WidgetRef ref, Category cat) {
    final hasProducts = cat.totalProducts > 0;
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        backgroundColor: AppColors.surface,
        shape:           RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
        title: Text(
          hasProducts ? '¿Desactivar categoría?' : '¿Eliminar categoría?',
          style: const TextStyle(color: AppColors.textPrimary),
        ),
        content: Text(
          hasProducts
              ? '"${cat.name}" tiene ${cat.totalProducts} producto(s). Se desactivará en lugar de eliminarse.'
              : '"${cat.name}" se eliminará permanentemente.',
          style: const TextStyle(color: AppColors.textSecondary),
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('Cancelar'),
          ),
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              if (hasProducts) {
                ref.read(categoriesAdminProvider.notifier).toggleActive(cat.id, false);
              } else {
                ref.read(categoriesAdminProvider.notifier).deleteCategory(cat.id);
              }
            },
            child: Text(
              hasProducts ? 'Desactivar' : 'Eliminar',
              style: const TextStyle(color: AppColors.error, fontWeight: FontWeight.bold),
            ),
          ),
        ],
      ),
    );
  }
}

// ── CategoryCard ──────────────────────────────────────────────

class _CategoryCard extends StatelessWidget {
  final Category     category;
  final VoidCallback onToggle;
  final VoidCallback onEdit;
  final VoidCallback onDelete;

  const _CategoryCard({
    required this.category,
    required this.onToggle,
    required this.onEdit,
    required this.onDelete,
  });

  @override
  Widget build(BuildContext context) => Opacity(
    opacity: category.isActive ? 1.0 : 0.55,
    child:   Container(
      padding:    const EdgeInsets.symmetric(horizontal: 12, vertical: 10),
      decoration: BoxDecoration(
        color:        AppColors.surface,
        borderRadius: BorderRadius.circular(14),
        border:       Border.all(color: AppColors.border),
      ),
      child: Row(
        children: [
          // Toggle
          Switch(
            value:       category.isActive,
            onChanged:   (_) => onToggle(),
            activeColor: AppColors.accent,
            trackColor:  WidgetStateProperty.resolveWith((s) =>
              s.contains(WidgetState.selected)
                ? AppColors.accent.withValues(alpha: 0.4)
                : AppColors.border,
            ),
          ),

          // Info
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Row(
                  children: [
                    Flexible(
                      child: Text(
                        category.name,
                        style: const TextStyle(
                          color: AppColors.textPrimary, fontWeight: FontWeight.w600,
                        ),
                        overflow: TextOverflow.ellipsis,
                      ),
                    ),
                    if (!category.isActive) ...[
                      const SizedBox(width: 6),
                      Container(
                        padding:    const EdgeInsets.symmetric(horizontal: 6, vertical: 1),
                        decoration: BoxDecoration(
                          color:        AppColors.error.withValues(alpha: 0.12),
                          borderRadius: BorderRadius.circular(6),
                        ),
                        child: const Text(
                          'Inactiva',
                          style: TextStyle(
                            color: AppColors.error, fontSize: 10, fontWeight: FontWeight.bold,
                          ),
                        ),
                      ),
                    ],
                  ],
                ),
                Text(
                  '/${category.slug}',
                  style: const TextStyle(
                    color: AppColors.textFaint, fontSize: 11, fontFamily: 'monospace',
                  ),
                ),
                Text(
                  '${category.totalProducts} producto${category.totalProducts != 1 ? "s" : ""}',
                  style: const TextStyle(
                    color: AppColors.accent, fontSize: 12, fontWeight: FontWeight.w600,
                  ),
                ),
              ],
            ),
          ),

          // Acciones
          Row(
            mainAxisSize: MainAxisSize.min,
            children: [
              IconButton(
                onPressed:   onEdit,
                icon:        const Icon(Icons.edit_outlined, size: 20),
                color:       AppColors.textSecondary,
                padding:     EdgeInsets.zero,
                constraints: const BoxConstraints(minWidth: 36, minHeight: 36),
              ),
              IconButton(
                onPressed:   onDelete,
                icon:        const Icon(Icons.delete_outline, size: 20),
                color:       AppColors.error,
                padding:     EdgeInsets.zero,
                constraints: const BoxConstraints(minWidth: 36, minHeight: 36),
              ),
            ],
          ),
        ],
      ),
    ),
  );
}
```

---

## 8.4 Actualizar el router — ruta `admin/categories`

```dart
// lib/presentation/navigation/app_router.dart — reemplazar el placeholder de categorías

import '../../presentation/screens/admin/categories_admin_screen.dart';

// Dentro del GoRoute de /admin/categories:
GoRoute(
  path:    '/admin/categories',
  builder: (_, state) => AdminShell(
    title:        'Categorías',
    currentRoute: state.matchedLocation,
    child:        const CategoriesAdminScreen(),
  ),
),
```

---

## ✅ Checkpoint Módulo 8

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Drawer → "Categorías" | Lista con toggle y acciones |
| 2 | Buscar "electro" | Filtrado local instantáneo |
| 3 | Toggle `Switch` | Activar/desactivar al instante (optimista) |
| 4 | Toggle sin conexión | Revierte al estado anterior |
| 5 | Botón "+ Nueva" | BottomSheet con formulario |
| 6 | Escribir nombre | Slug se auto-genera en el campo |
| 7 | Editar slug manualmente | Se fija y no cambia al editar nombre |
| 8 | Crear categoría válida | Aparece al inicio de la lista |
| 9 | Ícono ✏️ editar | Sheet con datos precargados |
| 10 | Eliminar con productos | Diálogo: "¿Desactivar?" |
| 11 | Eliminar sin productos | Diálogo: "¿Eliminar permanentemente?" |
| 12 | Verificar en catálogo público | La categoría activa/inactiva refleja el cambio |

---

## Resumen

| Elemento | Estado |
|---|---|
| `CategoriesAdminNotifier` con toggle optimista | ✅ |
| `filtered` getter local para búsqueda | ✅ |
| `CategoryFormState` sealed class | ✅ |
| `CategoryFormSheet` en BottomSheet Material 3 | ✅ |
| Auto-generación de slug con `toSlug()` | ✅ |
| `Switch` con colores del design system | ✅ |
| `CategoriesAdminScreen` con búsqueda y lista | ✅ |
| `_CategoryCard` con toggle, editar, eliminar | ✅ |
| `AlertDialog` inteligente (desactivar vs. eliminar) | ✅ |
| Ruta `admin/categories` en el router | ✅ |

**Siguiente módulo →** M9: Admin CRUD Productos — lista, crear, editar, restock e imagen
