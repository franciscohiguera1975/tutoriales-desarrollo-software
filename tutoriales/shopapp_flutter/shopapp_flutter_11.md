# Flutter Shop App — Módulo 11
## Admin CRUD Usuarios — Lista, crear, editar, toggle staff y activar/desactivar

> **Objetivo:** Gestión completa de usuarios con búsqueda debounced, filtros por rol/estado, formulario con contraseña opcional y dos toggles optimistas independientes (staff y activo).
> **Checkpoint final:** crear un nuevo usuario staff, verificar que puede acceder al panel admin y gestionar roles de usuarios existentes.

---

## 11.1 Provider de usuarios admin — `presentation/providers/users_admin_provider.dart`

```dart
// lib/presentation/providers/users_admin_provider.dart

import 'dart:async';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../data/remote/api/user_remote_datasource.dart';
import '../../domain/model/user.dart';

enum UserRoleFilter { all, clients, staff, active, inactive }

extension UserRoleFilterLabel on UserRoleFilter {
  String get label => switch (this) {
    UserRoleFilter.all      => 'Todos',
    UserRoleFilter.clients  => 'Clientes',
    UserRoleFilter.staff    => 'Staff',
    UserRoleFilter.active   => 'Activos',
    UserRoleFilter.inactive => 'Inactivos',
  };
}

class UsersAdminState {
  final List<User>    users;
  final bool          isLoading;
  final String?       error;
  final int           total;
  final String        search;
  final UserRoleFilter roleFilter;

  const UsersAdminState({
    this.users      = const [],
    this.isLoading  = false,
    this.error,
    this.total      = 0,
    this.search     = '',
    this.roleFilter = UserRoleFilter.all,
  });

  List<User> get filtered => users.where((u) {
    final matchSearch = search.isEmpty ||
        u.username.toLowerCase().contains(search.toLowerCase()) ||
        u.email.toLowerCase().contains(search.toLowerCase());

    final matchRole = switch (roleFilter) {
      UserRoleFilter.all      => true,
      UserRoleFilter.clients  => !u.isStaff,
      UserRoleFilter.staff    => u.isStaff,
      UserRoleFilter.active   => u.isActive,
      UserRoleFilter.inactive => !u.isActive,
    };
    return matchSearch && matchRole;
  }).toList();

  UsersAdminState copyWith({
    List<User>?    users,
    bool?          isLoading,
    String?        error,
    int?           total,
    String?        search,
    UserRoleFilter? roleFilter,
  }) => UsersAdminState(
    users:      users      ?? this.users,
    isLoading:  isLoading  ?? this.isLoading,
    error:      error,
    total:      total      ?? this.total,
    search:     search     ?? this.search,
    roleFilter: roleFilter ?? this.roleFilter,
  );
}

sealed class UserFormState { const UserFormState(); }
class UserFormIdle    extends UserFormState { const UserFormIdle(); }
class UserFormSaving  extends UserFormState { const UserFormSaving(); }
class UserFormSuccess extends UserFormState {
  final String message;
  const UserFormSuccess(this.message);
}
class UserFormError extends UserFormState {
  final String message;
  const UserFormError(this.message);
}

class UsersAdminNotifier extends StateNotifier<UsersAdminState> {
  final UserRemoteDatasource _datasource;

  UsersAdminNotifier(this._datasource) : super(const UsersAdminState()) {
    load();
  }

  final _formState = StateController<UserFormState>(const UserFormIdle());
  UserFormState get formState => _formState.state;

  Future<void> load() async {
    state = state.copyWith(isLoading: true, error: null);
    try {
      final result = await _datasource.getUsers();
      state = state.copyWith(users: result.results, total: result.count, isLoading: false);
    } catch (e) {
      state = state.copyWith(
        isLoading: false,
        error:     e.toString().replaceAll('Exception: ', ''),
      );
    }
  }

  void setSearch(String q)            => state = state.copyWith(search: q);
  void setRoleFilter(UserRoleFilter f) => state = state.copyWith(roleFilter: f);

  // Toggle staff — optimista
  void toggleStaff(int id, bool isStaff) {
    state = state.copyWith(
      users: state.users.map((u) =>
        u.id == id ? u.copyWith(isStaff: isStaff) : u,
      ).toList(),
    );
    _datasource.updateUser(id, {
      ...state.users.firstWhere((u) => u.id == id).toJson(),
      'is_staff': isStaff,
    }).catchError((_) {
      state = state.copyWith(
        users: state.users.map((u) =>
          u.id == id ? u.copyWith(isStaff: !isStaff) : u,
        ).toList(),
      );
    });
  }

  // Toggle activo — optimista con confirmación del servidor
  void toggleActive(int id) {
    final user = state.users.firstWhere((u) => u.id == id, orElse: () => state.users.first);
    final next = !user.isActive;
    state = state.copyWith(
      users: state.users.map((u) =>
        u.id == id ? u.copyWith(isActive: next) : u,
      ).toList(),
    );
    _datasource.toggleActive(id).then((serverActive) {
      state = state.copyWith(
        users: state.users.map((u) =>
          u.id == id ? u.copyWith(isActive: serverActive) : u,
        ).toList(),
      );
    }).catchError((_) {
      state = state.copyWith(
        users: state.users.map((u) =>
          u.id == id ? u.copyWith(isActive: !next) : u,
        ).toList(),
      );
    });
  }

  Future<void> createUser(Map<String, dynamic> payload) async {
    _formState.state = const UserFormSaving();
    try {
      final created = await _datasource.createUser(payload);
      state = state.copyWith(users: [created, ...state.users], total: state.total + 1);
      _formState.state = const UserFormSuccess('Usuario creado');
    } catch (e) {
      _formState.state = UserFormError(e.toString().replaceAll('Exception: ', ''));
    }
  }

  Future<void> updateUser(int id, Map<String, dynamic> payload) async {
    _formState.state = const UserFormSaving();
    try {
      final updated = await _datasource.updateUser(id, payload);
      state = state.copyWith(
        users: state.users.map((u) => u.id == id ? updated : u).toList(),
      );
      _formState.state = const UserFormSuccess('Usuario actualizado');
    } catch (e) {
      _formState.state = UserFormError(e.toString().replaceAll('Exception: ', ''));
    }
  }

  Future<void> deleteUser(int id) async {
    try {
      await _datasource.deleteUser(id);
      state = state.copyWith(
        users: state.users.where((u) => u.id != id).toList(),
        total: state.total - 1,
      );
    } catch (e) {
      state = state.copyWith(error: e.toString().replaceAll('Exception: ', ''));
    }
  }

  void resetFormState() => _formState.state = const UserFormIdle();
}

final usersAdminProvider =
    StateNotifierProvider<UsersAdminNotifier, UsersAdminState>((ref) {
  return UsersAdminNotifier(ref.watch(userDatasourceProvider));
});
```

---

## 11.2 Formulario de usuario — `presentation/widgets/user_form.dart`

```dart
// lib/presentation/widgets/user_form.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/validators.dart';
import '../../domain/model/user.dart';
import '../providers/users_admin_provider.dart';

Future<void> showUserForm(
  BuildContext context,
  WidgetRef    ref, {
  User? initial,
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
      child:  UserFormSheet(initial: initial),
    ),
  );
}

class UserFormSheet extends ConsumerStatefulWidget {
  final User? initial;
  const UserFormSheet({super.key, this.initial});

  @override
  ConsumerState<UserFormSheet> createState() => _UserFormSheetState();
}

class _UserFormSheetState extends ConsumerState<UserFormSheet> {
  final _formKey    = GlobalKey<FormState>();
  final _userCtrl   = TextEditingController();
  final _emailCtrl  = TextEditingController();
  final _fnCtrl     = TextEditingController();
  final _lnCtrl     = TextEditingController();
  final _passCtrl   = TextEditingController();
  bool  _isStaff    = false;
  bool  _isActive   = true;
  bool  _showPass   = false;

  @override
  void initState() {
    super.initState();
    if (widget.initial != null) {
      final u       = widget.initial!;
      _userCtrl.text  = u.username;
      _emailCtrl.text = u.email;
      _fnCtrl.text    = u.firstName;
      _lnCtrl.text    = u.lastName;
      _isStaff        = u.isStaff;
      _isActive       = u.isActive;
    }
  }

  @override
  void dispose() {
    _userCtrl.dispose(); _emailCtrl.dispose();
    _fnCtrl.dispose();   _lnCtrl.dispose();
    _passCtrl.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    if (!_formKey.currentState!.validate()) return;
    final payload = {
      'username':   _userCtrl.text.trim(),
      'email':      _emailCtrl.text.trim(),
      'first_name': _fnCtrl.text.trim(),
      'last_name':  _lnCtrl.text.trim(),
      'is_staff':   _isStaff,
      'is_active':  _isActive,
      if (_passCtrl.text.isNotEmpty) 'password': _passCtrl.text,
    };
    if (widget.initial != null) {
      await ref.read(usersAdminProvider.notifier)
          .updateUser(widget.initial!.id, payload);
    } else {
      await ref.read(usersAdminProvider.notifier).createUser(payload);
    }
  }

  @override
  Widget build(BuildContext context) {
    final notifier = ref.read(usersAdminProvider.notifier);
    final formSt   = notifier.formState;
    final isSaving = formSt is UserFormSaving;
    final isEdit   = widget.initial != null;

    if (formSt is UserFormSuccess) {
      WidgetsBinding.instance.addPostFrameCallback((_) {
        if (mounted) Navigator.pop(context);
      });
    }

    return Padding(
      padding: EdgeInsets.only(bottom: MediaQuery.of(context).viewInsets.bottom),
      child:   SingleChildScrollView(
        padding: const EdgeInsets.fromLTRB(24, 8, 24, 32),
        child:   Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
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
              isEdit ? 'Editar: ${widget.initial!.username}' : 'Nuevo usuario',
              style: const TextStyle(
                color: AppColors.textPrimary, fontSize: 20, fontWeight: FontWeight.bold,
              ),
            ),
            const SizedBox(height: 20),

            if (formSt is UserFormError) ...[
              Container(
                width:   double.infinity,
                padding: const EdgeInsets.all(12),
                decoration: BoxDecoration(
                  color:        AppColors.error.withValues(alpha: 0.1),
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
                  // Usuario y Email
                  Row(
                    children: [
                      Expanded(
                        child: TextFormField(
                          controller: _userCtrl, enabled: !isSaving,
                          decoration: const InputDecoration(labelText: 'Usuario *'),
                          style:      const TextStyle(color: AppColors.textPrimary),
                          validator:  validateUsername,
                        ),
                      ),
                      const SizedBox(width: 12),
                      Expanded(
                        child: TextFormField(
                          controller:  _emailCtrl, enabled: !isSaving,
                          keyboardType:TextInputType.emailAddress,
                          decoration:  const InputDecoration(labelText: 'Email *'),
                          style:       const TextStyle(color: AppColors.textPrimary),
                          validator:   validateEmail,
                        ),
                      ),
                    ],
                  ),
                  const SizedBox(height: 12),

                  // Nombre y Apellido
                  Row(
                    children: [
                      Expanded(
                        child: TextFormField(
                          controller: _fnCtrl, enabled: !isSaving,
                          decoration: const InputDecoration(labelText: 'Nombre'),
                          style:      const TextStyle(color: AppColors.textPrimary),
                        ),
                      ),
                      const SizedBox(width: 12),
                      Expanded(
                        child: TextFormField(
                          controller: _lnCtrl, enabled: !isSaving,
                          decoration: const InputDecoration(labelText: 'Apellido'),
                          style:      const TextStyle(color: AppColors.textPrimary),
                        ),
                      ),
                    ],
                  ),
                  const SizedBox(height: 12),

                  // Contraseña
                  TextFormField(
                    controller:    _passCtrl,
                    obscureText:   !_showPass,
                    enabled:       !isSaving,
                    decoration:    InputDecoration(
                      labelText: isEdit
                          ? 'Nueva contraseña (vacío = no cambiar)'
                          : 'Contraseña *',
                      suffixIcon: IconButton(
                        icon:  Icon(
                          _showPass ? Icons.visibility_off_outlined : Icons.visibility_outlined,
                          color: AppColors.textSecondary, size: 20,
                        ),
                        onPressed: () => setState(() => _showPass = !_showPass),
                      ),
                    ),
                    style:   const TextStyle(color: AppColors.textPrimary),
                    validator: isEdit
                        ? null
                        : (v) => (v == null || v.isEmpty)
                            ? 'Contraseña obligatoria'
                            : (v.length < 8 ? 'Mínimo 8 caracteres' : null),
                  ),
                  const SizedBox(height: 12),

                  // Toggles Staff y Activo en fila
                  Row(
                    children: [
                      Expanded(child: _ToggleCard(
                        label:       'Rol Staff',
                        description: 'Acceso al admin',
                        value:       _isStaff,
                        onChanged:   isSaving ? null : (v) => setState(() => _isStaff = v),
                      )),
                      const SizedBox(width: 12),
                      Expanded(child: _ToggleCard(
                        label:       'Activo',
                        description: 'Puede iniciar sesión',
                        value:       _isActive,
                        onChanged:   isSaving ? null : (v) => setState(() => _isActive = v),
                      )),
                    ],
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
                              : Text(isEdit ? 'Guardar cambios' : 'Crear usuario'),
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

class _ToggleCard extends StatelessWidget {
  final String       label;
  final String       description;
  final bool         value;
  final ValueChanged<bool>? onChanged;

  const _ToggleCard({
    required this.label,
    required this.description,
    required this.value,
    this.onChanged,
  });

  @override
  Widget build(BuildContext context) => Container(
    padding:    const EdgeInsets.symmetric(horizontal: 12, vertical: 10),
    decoration: BoxDecoration(
      color:        AppColors.surface2,
      borderRadius: BorderRadius.circular(12),
    ),
    child: Row(
      mainAxisAlignment: MainAxisAlignment.spaceBetween,
      children: [
        Expanded(
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.start,
            children: [
              Text(label,
                  style: const TextStyle(
                    color: AppColors.textPrimary, fontWeight: FontWeight.w600, fontSize: 13,
                  )),
              Text(description,
                  style: const TextStyle(color: AppColors.textSecondary, fontSize: 10)),
            ],
          ),
        ),
        Switch(
          value:       value,
          onChanged:   onChanged,
          activeColor: AppColors.accent,
        ),
      ],
    ),
  );
}
```

---

## 11.3 Pantalla de usuarios admin — `presentation/screens/admin/users_admin_screen.dart`

```dart
// lib/presentation/screens/admin/users_admin_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/formatters.dart';
import '../../../domain/model/user.dart';
import '../providers/users_admin_provider.dart';
import '../widgets/user_form.dart';

class UsersAdminScreen extends ConsumerWidget {
  const UsersAdminScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state    = ref.watch(usersAdminProvider);
    final filtered = state.filtered;
    final tt       = Theme.of(context).textTheme;

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
                      Text('Usuarios', style: tt.headlineMedium),
                      Text('${state.total} usuarios',
                          style: const TextStyle(color: AppColors.textSecondary, fontSize: 13)),
                    ],
                  ),
                  Row(
                    children: [
                      IconButton(
                        onPressed: () => ref.read(usersAdminProvider.notifier).load(),
                        icon: const Icon(Icons.refresh_rounded, color: AppColors.textSecondary),
                      ),
                      ElevatedButton.icon(
                        onPressed: () => showUserForm(context, ref),
                        icon:      const Icon(Icons.person_add_outlined, size: 18),
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
                onChanged:  ref.read(usersAdminProvider.notifier).setSearch,
                decoration: const InputDecoration(
                  hintText:   'Buscar usuario o email...',
                  prefixIcon: Icon(Icons.search_rounded, color: AppColors.textSecondary),
                  contentPadding: EdgeInsets.symmetric(vertical: 10),
                ),
                style: const TextStyle(color: AppColors.textPrimary),
              ),
              const SizedBox(height: 10),

              // Chips de filtro de rol
              SizedBox(
                height: 34,
                child:  ListView(
                  scrollDirection: Axis.horizontal,
                  children: UserRoleFilter.values.map((f) => Padding(
                    padding: const EdgeInsets.only(right: 8),
                    child:   ChoiceChip(
                      label:     Text(f.label),
                      selected:  state.roleFilter == f,
                      onSelected:(_) =>
                          ref.read(usersAdminProvider.notifier).setRoleFilter(f),
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
                      onPressed: () => ref.read(usersAdminProvider.notifier).load(),
                      child: const Text('Reintentar'),
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
                    Text('👤', style: TextStyle(fontSize: 48)),
                    SizedBox(height: 12),
                    Text('Sin usuarios',
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
              itemBuilder: (_, i) => _UserCard(
                user:          filtered[i],
                onToggleStaff: () => ref.read(usersAdminProvider.notifier)
                    .toggleStaff(filtered[i].id, !filtered[i].isStaff),
                onToggleActive:() => ref.read(usersAdminProvider.notifier)
                    .toggleActive(filtered[i].id),
                onEdit:        () => showUserForm(context, ref, initial: filtered[i]),
                onDelete:      () => _confirmDelete(context, ref, filtered[i]),
              ),
            );
          }),
        ),
      ],
    );
  }

  void _confirmDelete(BuildContext context, WidgetRef ref, User user) {
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        backgroundColor: AppColors.surface,
        shape:           RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
        title:           const Text('¿Eliminar usuario?',
            style: TextStyle(color: AppColors.textPrimary)),
        content:         Text(
          '"${user.username}" se eliminará permanentemente.',
          style: const TextStyle(color: AppColors.textSecondary),
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child:     const Text('Cancelar'),
          ),
          TextButton(
            onPressed: () {
              Navigator.pop(context);
              ref.read(usersAdminProvider.notifier).deleteUser(user.id);
            },
            child: const Text('Eliminar',
                style: TextStyle(color: AppColors.error, fontWeight: FontWeight.bold)),
          ),
        ],
      ),
    );
  }
}

// ── UserCard ──────────────────────────────────────────────────

class _UserCard extends StatelessWidget {
  final User         user;
  final VoidCallback onToggleStaff;
  final VoidCallback onToggleActive;
  final VoidCallback onEdit;
  final VoidCallback onDelete;

  const _UserCard({
    required this.user,
    required this.onToggleStaff,
    required this.onToggleActive,
    required this.onEdit,
    required this.onDelete,
  });

  @override
  Widget build(BuildContext context) => Opacity(
    opacity: user.isActive ? 1.0 : 0.55,
    child:   Container(
      padding:    const EdgeInsets.all(12),
      decoration: BoxDecoration(
        color:        AppColors.surface,
        borderRadius: BorderRadius.circular(14),
        border:       Border.all(color: AppColors.border),
      ),
      child: Row(
        children: [
          // Avatar
          Container(
            width:  46, height: 46,
            decoration: BoxDecoration(
              gradient: user.isStaff
                  ? const LinearGradient(colors: [AppColors.accent, AppColors.accentLight])
                  : LinearGradient(colors: [AppColors.surface2, AppColors.border]),
              shape: BoxShape.circle,
            ),
            child: Center(
              child: Text(
                user.username.isNotEmpty ? user.username[0].toUpperCase() : '?',
                style: TextStyle(
                  color:      user.isStaff ? AppColors.onAccent : AppColors.textSecondary,
                  fontWeight: FontWeight.bold, fontSize: 18,
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
                Row(
                  children: [
                    Flexible(
                      child: Text(
                        user.username,
                        style: const TextStyle(
                          color: AppColors.textPrimary, fontWeight: FontWeight.w600,
                        ),
                        overflow: TextOverflow.ellipsis,
                      ),
                    ),
                    const SizedBox(width: 6),
                    if (user.isStaff)
                      Container(
                        padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 1),
                        decoration: BoxDecoration(
                          color:        AppColors.accent.withValues(alpha: 0.15),
                          borderRadius: BorderRadius.circular(6),
                        ),
                        child: const Text('Staff',
                            style: TextStyle(
                              color: AppColors.accent, fontSize: 10, fontWeight: FontWeight.bold,
                            )),
                      ),
                    if (!user.isActive) ...[
                      const SizedBox(width: 4),
                      Container(
                        padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 1),
                        decoration: BoxDecoration(
                          color:        AppColors.error.withValues(alpha: 0.12),
                          borderRadius: BorderRadius.circular(6),
                        ),
                        child: const Text('Inactivo',
                            style: TextStyle(
                              color: AppColors.error, fontSize: 10, fontWeight: FontWeight.bold,
                            )),
                      ),
                    ],
                  ],
                ),
                Text(user.email,
                    style: const TextStyle(color: AppColors.textSecondary, fontSize: 12),
                    overflow: TextOverflow.ellipsis),
                Text(
                  '${user.numOrders} pedido${user.numOrders != 1 ? "s" : ""}',
                  style: const TextStyle(
                    color: AppColors.accent, fontSize: 11, fontWeight: FontWeight.w600,
                  ),
                ),
              ],
            ),
          ),

          // Acciones
          Column(
            crossAxisAlignment: CrossAxisAlignment.end,
            children: [
              Row(
                mainAxisSize: MainAxisSize.min,
                children: [
                  // Toggle staff
                  GestureDetector(
                    onTap: onToggleStaff,
                    child: Padding(
                      padding: const EdgeInsets.all(4),
                      child:   Icon(
                        user.isStaff
                            ? Icons.admin_panel_settings
                            : Icons.person_outline,
                        color:  user.isStaff ? AppColors.accent : AppColors.textFaint,
                        size:   20,
                      ),
                    ),
                  ),
                  // Toggle activo
                  GestureDetector(
                    onTap: onToggleActive,
                    child: Padding(
                      padding: const EdgeInsets.all(4),
                      child:   Icon(
                        user.isActive ? Icons.toggle_on : Icons.toggle_off,
                        color:  user.isActive ? AppColors.success : AppColors.textFaint,
                        size:   26,
                      ),
                    ),
                  ),
                  // Editar
                  GestureDetector(
                    onTap: onEdit,
                    child: const Padding(
                      padding: EdgeInsets.all(4),
                      child:   Icon(Icons.edit_outlined,
                          color: AppColors.textSecondary, size: 20),
                    ),
                  ),
                  // Eliminar
                  GestureDetector(
                    onTap: onDelete,
                    child: const Padding(
                      padding: EdgeInsets.all(4),
                      child:   Icon(Icons.person_remove_outlined,
                          color: AppColors.error, size: 20),
                    ),
                  ),
                ],
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

## 11.4 Router final completo — `presentation/navigation/app_router.dart`

```dart
// lib/presentation/navigation/app_router.dart — versión final completa

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';

import '../../domain/model/auth_state.dart';
import '../providers/auth_provider.dart';
import '../screens/auth/login_screen.dart';
import '../screens/auth/register_screen.dart';
import '../screens/auth/profile_screen.dart';
import '../screens/catalog/home_screen.dart';
import '../screens/catalog/catalog_screen.dart';
import '../screens/catalog/product_detail_screen.dart';
import '../screens/cart/cart_screen.dart';
import '../screens/orders/orders_screen.dart';
import '../screens/orders/order_detail_screen.dart';
import '../screens/admin/dashboard_screen.dart';
import '../widgets/admin_shell.dart';
import '../screens/admin/categories_admin_screen.dart';
import '../screens/admin/products_admin_screen.dart';
import '../screens/admin/orders_admin_screen.dart';
import '../screens/admin/order_admin_detail_screen.dart';
import '../screens/admin/users_admin_screen.dart';
import 'public_shell.dart';

final routerProvider = Provider<GoRouter>((ref) {
  return GoRouter(
    initialLocation: '/',
    refreshListenable: _AuthStateListenable(ref),
    redirect: (context, state) {
      final auth     = ref.read(authProvider);
      final location = state.matchedLocation;

      if (auth.isChecking) return null;

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
          GoRoute(
            path: '/catalog',
            builder: (_, __) => const CatalogScreen(),
            routes: [
              GoRoute(
                path: ':id',
                builder: (_, s) => ProductDetailScreen(
                  productId: int.parse(s.pathParameters['id']!),
                ),
              ),
            ],
          ),
          GoRoute(path: '/cart',    builder: (_, __) => const CartScreen()),
          GoRoute(path: '/orders',  builder: (_, __) => const OrdersScreen()),
          GoRoute(
            path:    '/orders/:id',
            builder: (_, s) => OrderDetailScreen(
              orderId: int.parse(s.pathParameters['id']!),
            ),
          ),
          GoRoute(path: '/profile', builder: (_, __) => const ProfileScreen()),
        ],
      ),

      // ── Admin ─────────────────────────────────────────────
      GoRoute(
        path:    '/admin',
        builder: (_, s) => AdminShell(
          title: 'Dashboard', currentRoute: s.matchedLocation,
          child: const DashboardScreen(),
        ),
      ),
      GoRoute(
        path:    '/admin/categories',
        builder: (_, s) => AdminShell(
          title: 'Categorías', currentRoute: s.matchedLocation,
          child: const CategoriesAdminScreen(),
        ),
      ),
      GoRoute(
        path:    '/admin/products',
        builder: (_, s) => AdminShell(
          title: 'Productos', currentRoute: s.matchedLocation,
          child: const ProductsAdminScreen(),
        ),
      ),
      GoRoute(
        path:    '/admin/orders',
        builder: (_, s) => AdminShell(
          title: 'Pedidos', currentRoute: s.matchedLocation,
          child: const OrdersAdminScreen(),
        ),
      ),
      GoRoute(
        path:    '/admin/orders/:id',
        builder: (_, s) => AdminShell(
          title: 'Detalle pedido #${s.pathParameters['id']}',
          currentRoute: '/admin/orders',
          child: OrderAdminDetailScreen(
            orderId: int.parse(s.pathParameters['id']!),
          ),
        ),
      ),
      GoRoute(
        path:    '/admin/users',
        builder: (_, s) => AdminShell(
          title: 'Usuarios', currentRoute: s.matchedLocation,
          child: const UsersAdminScreen(),
        ),
      ),
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

## ✅ Checkpoint Módulo 11 — Final

### Verificar en Django

```bash
uv run python manage.py shell -c "
from django.contrib.auth.models import User
users = User.objects.all()
for u in users:
    print(f'{u.username} staff={u.is_staff} active={u.is_active}')
"
```

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Drawer → "Usuarios" | Lista con avatares dorados (staff) y grises |
| 2 | Filtro "Staff" | Solo usuarios con rol staff |
| 3 | Filtro "Inactivos" | Usuarios con opacidad reducida |
| 4 | Buscar "admin" | Filtrado local instantáneo |
| 5 | Ícono 🛡️ toggle staff | Avatar cambia a dorado |
| 6 | Ícono toggle activo | Badge "Inactivo" aparece |
| 7 | Botón "+ Nuevo" | BottomSheet con formulario |
| 8 | Email sin @ | Error de validación inline |
| 9 | Crear usuario con is_staff=true | Badge dorado en la lista |
| 10 | Nuevo staff hace login | Redirige a `/admin` automáticamente |
| 11 | Editar contraseña vacía | No cambia la contraseña existente |
| 12 | Eliminar usuario | Desaparece de la lista |

---

## Resumen del tutorial completo — 11 módulos Flutter

| Módulo | Contenido | Líneas |
|--------|-----------|--------|
| M1  | Setup + Material 3 + modelos de dominio | 1174 |
| M2  | Dio + interceptores JWT + datasources | 990 |
| M3  | Auth — AuthNotifier + Login + Registro | 925 |
| M4  | GoRouter + ShellRoute + Home + Catálogo | 1034 |
| M5  | Detalle + CartBottomSheet + Checkout | 1143 |
| M6  | Pedidos + progreso + Perfil + Logout | 1083 |
| M7  | NavigationDrawer + Dashboard con KPIs | 914 |
| M8  | Admin CRUD Categorías | 787 |
| M9  | Admin CRUD Productos + restock | 1019 |
| M10 | Admin CRUD Pedidos + estados | 890 |
| M11 | Admin CRUD Usuarios + router final | ~1100 |

### Endpoints consumidos — resumen completo

```
POST   /api/auth/login/                  → M3
POST   /api/auth/register/               → M3
POST   /api/auth/logout/                 → M6
POST   /api/auth/token/refresh/          → M2 (interceptor)
GET    /api/categories/                  → M4, M7, M8, M9
POST   /api/categories/                  → M8
PATCH  /api/categories/{id}/             → M8
DELETE /api/categories/{id}/             → M8
GET    /api/categories/stats/            → M7
GET    /api/products/                    → M4, M7, M9
GET    /api/products/{id}/               → M5
POST   /api/products/                    → M9
PATCH  /api/products/{id}/               → M9
DELETE /api/products/{id}/               → M9
POST   /api/products/{id}/restock/       → M9
GET    /api/products/stats/              → M7
POST   /api/orders/                      → M5
POST   /api/orders/{id}/add-item/        → M5
POST   /api/orders/{id}/confirm/         → M5
GET    /api/orders/                      → M6, M10
GET    /api/orders/{id}/                 → M6, M10
POST   /api/orders/{id}/update-status/   → M10
GET    /api/orders/stats/                → M7
GET    /api/users/                       → M11
POST   /api/users/                       → M11
PATCH  /api/users/{id}/                  → M11
DELETE /api/users/{id}/                  → M11
POST   /api/users/{id}/toggle-active/    → M11
GET    /api/users/stats/                 → M7
```
