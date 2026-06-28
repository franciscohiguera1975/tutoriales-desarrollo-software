# Flutter Shop App — Módulo 3
## Auth — Login, Registro, AuthNotifier y persistencia de sesión

> **Objetivo:** Sistema de autenticación completo con Riverpod: datasource de auth, AuthNotifier con estado reactivo, pantallas de Login y Registro, y restauración de sesión al arrancar la app.
> **Checkpoint final:** login con credenciales reales, sesión persistida al cerrar y reabrir la app, redirección automática según el rol (cliente → Home, staff → Admin).

---

## 3.1 Datasource de auth — `data/remote/api/auth_remote_datasource.dart`

```dart
// lib/data/remote/api/auth_remote_datasource.dart

import 'package:dio/dio.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../../core/error/api_exception.dart';
import 'dio_client.dart';
import '../../local/secure_storage.dart';
import '../../../domain/model/auth_models.dart';

abstract class AuthRemoteDatasource {
  Future<LoggedUser> login(String username, String password);
  Future<LoggedUser> register(String username, String email, String password, String password2);
  Future<void>       logout();
}

class AuthRemoteDatasourceImpl implements AuthRemoteDatasource {
  final Dio           _dio;
  final SecureStorage _storage;

  AuthRemoteDatasourceImpl(this._dio, this._storage);

  @override
  Future<LoggedUser> login(String username, String password) async {
    try {
      final res  = await _dio.post(
        '/auth/login/',
        data: {'username': username, 'password': password},
      );
      final data = res.data as Map<String, dynamic>;
      await _storage.saveTokens(data['access'] as String, data['refresh'] as String);
      await _storage.saveUser(
        id:       data['user_id'] as int,
        username: data['username'] as String,
        email:    data['email']    as String,
        isStaff:  data['is_staff'] as bool,
      );
      return LoggedUser.fromMap(data);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<LoggedUser> register(
    String username,
    String email,
    String password,
    String password2,
  ) async {
    try {
      final res = await _dio.post(
        '/auth/register/',
        data: {
          'username':  username,
          'email':     email,
          'password':  password,
          'password2': password2,
        },
      );
      final data = res.data as Map<String, dynamic>;
      await _storage.saveTokens(data['access'] as String, data['refresh'] as String);
      await _storage.saveUser(
        id:       data['user_id'] as int,
        username: data['username'] as String,
        email:    data['email']    as String,
        isStaff:  data['is_staff'] as bool,
      );
      return LoggedUser.fromMap(data);
    } on DioException catch (e) {
      throw ApiException.fromDioError(e);
    }
  }

  @override
  Future<void> logout() async {
    try {
      final refresh = await _storage.getRefresh();
      if (refresh != null && refresh.isNotEmpty) {
        await _dio.post('/auth/logout/', data: {'refresh': refresh});
      }
    } catch (_) {
      // Si el logout falla en el servidor, limpiamos localmente igual
    } finally {
      await _storage.clearSession();
    }
  }
}

final authDatasourceProvider = Provider<AuthRemoteDatasource>((ref) {
  return AuthRemoteDatasourceImpl(
    ref.watch(dioProvider),
    ref.watch(secureStorageProvider),
  );
});
```

---

## 3.2 Estado de Auth — `domain/model/auth_state.dart`

```dart
// lib/domain/model/auth_state.dart

import 'auth_models.dart';

enum AuthStatus { checking, authenticated, unauthenticated }

class AuthState {
  final AuthStatus status;
  final LoggedUser? user;
  final String?     error;

  const AuthState({
    required this.status,
    this.user,
    this.error,
  });

  const AuthState.checking()
      : status = AuthStatus.checking,
        user   = null,
        error  = null;

  const AuthState.authenticated(this.user)
      : status = AuthStatus.authenticated,
        error  = null;

  const AuthState.unauthenticated([this.error])
      : status = AuthStatus.unauthenticated,
        user   = null;

  bool get isAuthenticated  => status == AuthStatus.authenticated;
  bool get isChecking       => status == AuthStatus.checking;
  bool get isStaff          => user?.isStaff ?? false;
  bool get isUnauthenticated=> status == AuthStatus.unauthenticated;

  AuthState copyWith({AuthStatus? status, LoggedUser? user, String? error}) => AuthState(
    status: status ?? this.status,
    user:   user   ?? this.user,
    error:  error,
  );
}
```

---

## 3.3 AuthNotifier — `presentation/providers/auth_provider.dart`

```dart
// lib/presentation/providers/auth_provider.dart

import 'package:flutter_riverpod/flutter_riverpod.dart';
import '../../core/error/api_exception.dart';
import '../../data/local/secure_storage.dart';
import '../../data/remote/api/auth_remote_datasource.dart';
import '../../domain/model/auth_models.dart';
import '../../domain/model/auth_state.dart';

class AuthNotifier extends StateNotifier<AuthState> {
  final AuthRemoteDatasource _datasource;
  final SecureStorage        _storage;

  AuthNotifier(this._datasource, this._storage) : super(const AuthState.checking()) {
    _restoreSession();
  }

  // Restaurar sesión al iniciar la app
  Future<void> _restoreSession() async {
    try {
      final isLoggedIn = await _storage.isLoggedIn();
      if (!isLoggedIn) {
        state = const AuthState.unauthenticated();
        return;
      }
      final userData = await _storage.getUser();
      if (userData == null) {
        state = const AuthState.unauthenticated();
        return;
      }
      final user = LoggedUser(
        id:       int.parse(userData['id']!),
        username: userData['username']!,
        email:    userData['email']!,
        isStaff:  userData['is_staff'] == 'true',
      );
      state = AuthState.authenticated(user);
    } catch (_) {
      state = const AuthState.unauthenticated();
    }
  }

  // Login
  Future<void> login(String username, String password) async {
    state = const AuthState.checking();
    try {
      final user = await _datasource.login(username.trim(), password);
      state = AuthState.authenticated(user);
    } on ApiException catch (e) {
      state = AuthState.unauthenticated(e.message);
    } catch (e) {
      state = const AuthState.unauthenticated('Error inesperado. Intenta de nuevo.');
    }
  }

  // Registro
  Future<void> register(
    String username,
    String email,
    String password,
    String password2,
  ) async {
    state = const AuthState.checking();
    try {
      final user = await _datasource.register(
        username.trim(), email.trim(), password, password2,
      );
      state = AuthState.authenticated(user);
    } on ApiException catch (e) {
      state = AuthState.unauthenticated(e.message);
    } catch (e) {
      state = const AuthState.unauthenticated('Error inesperado. Intenta de nuevo.');
    }
  }

  // Logout
  Future<void> logout() async {
    await _datasource.logout();
    state = const AuthState.unauthenticated();
  }

  void clearError() {
    if (state.isUnauthenticated && state.error != null) {
      state = const AuthState.unauthenticated();
    }
  }
}

final authProvider = StateNotifierProvider<AuthNotifier, AuthState>((ref) {
  return AuthNotifier(
    ref.watch(authDatasourceProvider),
    ref.watch(secureStorageProvider),
  );
});
```

---

## 3.4 Widgets compartidos de Auth

### `presentation/widgets/auth_text_field.dart`

```dart
// lib/presentation/widgets/auth_text_field.dart

import 'package:flutter/material.dart';
import '../../theme/app_colors.dart';

class AuthTextField extends StatefulWidget {
  final String         label;
  final String?        hint;
  final bool           isPassword;
  final bool           enabled;
  final String?        errorText;
  final TextInputType  keyboardType;
  final TextEditingController controller;
  final String? Function(String?)? validator;
  final void Function(String)? onChanged;
  final TextInputAction textInputAction;

  const AuthTextField({
    super.key,
    required this.label,
    required this.controller,
    this.hint,
    this.isPassword      = false,
    this.enabled         = true,
    this.errorText,
    this.keyboardType    = TextInputType.text,
    this.validator,
    this.onChanged,
    this.textInputAction = TextInputAction.next,
  });

  @override
  State<AuthTextField> createState() => _AuthTextFieldState();
}

class _AuthTextFieldState extends State<AuthTextField> {
  bool _obscure = true;

  @override
  Widget build(BuildContext context) {
    return TextFormField(
      controller:     widget.controller,
      obscureText:    widget.isPassword && _obscure,
      keyboardType:   widget.keyboardType,
      textInputAction:widget.textInputAction,
      enabled:        widget.enabled,
      onChanged:      widget.onChanged,
      validator:      widget.validator,
      style:          const TextStyle(color: AppColors.textPrimary),
      decoration:     InputDecoration(
        labelText:  widget.label,
        hintText:   widget.hint,
        errorText:  widget.errorText,
        suffixIcon: widget.isPassword
            ? IconButton(
                icon: Icon(
                  _obscure ? Icons.visibility_off_outlined : Icons.visibility_outlined,
                  color: AppColors.textSecondary,
                  size: 20,
                ),
                onPressed: () => setState(() => _obscure = !_obscure),
              )
            : null,
      ),
    );
  }
}
```

### `presentation/widgets/auth_button.dart`

```dart
// lib/presentation/widgets/auth_button.dart

import 'package:flutter/material.dart';
import '../../theme/app_colors.dart';

class AuthButton extends StatelessWidget {
  final String   label;
  final VoidCallback? onPressed;
  final bool     isLoading;

  const AuthButton({
    super.key,
    required this.label,
    required this.onPressed,
    this.isLoading = false,
  });

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width:  double.infinity,
      height: 52,
      child: ElevatedButton(
        onPressed: isLoading ? null : onPressed,
        child:     isLoading
            ? const SizedBox(
                width:  20,
                height: 20,
                child:  CircularProgressIndicator(
                  strokeWidth: 2.5,
                  color:       AppColors.onAccent,
                ),
              )
            : Text(label),
      ),
    );
  }
}
```

---

## 3.5 Pantalla Login — `presentation/screens/auth/login_screen.dart`

```dart
// lib/presentation/screens/auth/login_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/validators.dart';
import '../../../domain/model/auth_state.dart';
import '../../providers/auth_provider.dart';
import '../../widgets/auth_button.dart';
import '../../widgets/auth_text_field.dart';

class LoginScreen extends ConsumerStatefulWidget {
  const LoginScreen({super.key});

  @override
  ConsumerState<LoginScreen> createState() => _LoginScreenState();
}

class _LoginScreenState extends ConsumerState<LoginScreen> {
  final _formKey    = GlobalKey<FormState>();
  final _userCtrl   = TextEditingController();
  final _passCtrl   = TextEditingController();
  bool  _submitted  = false;

  @override
  void dispose() {
    _userCtrl.dispose();
    _passCtrl.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    setState(() => _submitted = true);
    if (!_formKey.currentState!.validate()) return;
    await ref.read(authProvider.notifier).login(
      _userCtrl.text,
      _passCtrl.text,
    );
  }

  @override
  Widget build(BuildContext context) {
    final authState = ref.watch(authProvider);
    final isLoading = authState.isChecking;
    final error     = authState.error;
    final tt        = Theme.of(context).textTheme;

    // Escuchar cambios de estado para navegar
    ref.listen<AuthState>(authProvider, (_, next) {
      if (next.isAuthenticated) {
        final dest = next.isStaff ? '/admin' : '/';
        context.go(dest);
      }
    });

    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.symmetric(horizontal: 24),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.center,
            children: [
              const SizedBox(height: 80),

              // Logo
              Text(
                'Flutter Shop App',
                style: tt.displayMedium?.copyWith(color: AppColors.accent),
              ),
              const SizedBox(height: 8),
              Text('Inicia sesión en tu cuenta', style: tt.bodyMedium),
              const SizedBox(height: 48),

              // Card del formulario
              Container(
                padding:    const EdgeInsets.all(24),
                decoration: BoxDecoration(
                  color:        AppColors.surface,
                  borderRadius: BorderRadius.circular(20),
                  border:       Border.all(color: AppColors.border),
                ),
                child: Form(
                  key: _formKey,
                  child: Column(
                    crossAxisAlignment: CrossAxisAlignment.start,
                    children: [

                      // Error general del servidor
                      if (error != null) ...[
                        Container(
                          width:   double.infinity,
                          padding: const EdgeInsets.all(12),
                          decoration: BoxDecoration(
                            color:        AppColors.error.withValues(alpha: 0.1),
                            borderRadius: BorderRadius.circular(10),
                            border:       Border.all(color: AppColors.error.withValues(alpha: 0.3)),
                          ),
                          child: Text(
                            error,
                            style: const TextStyle(color: AppColors.error, fontSize: 13),
                          ),
                        ),
                        const SizedBox(height: 16),
                      ],

                      // Campo usuario
                      AuthTextField(
                        label:      'Usuario',
                        hint:       'tu_usuario',
                        controller: _userCtrl,
                        enabled:    !isLoading,
                        validator:  _submitted ? validateUsername : null,
                        onChanged:  (_) => ref.read(authProvider.notifier).clearError(),
                      ),
                      const SizedBox(height: 16),

                      // Campo contraseña
                      AuthTextField(
                        label:           'Contraseña',
                        hint:            '••••••••',
                        controller:      _passCtrl,
                        isPassword:      true,
                        enabled:         !isLoading,
                        keyboardType:    TextInputType.visiblePassword,
                        textInputAction: TextInputAction.done,
                        validator:       _submitted
                            ? (v) => (v == null || v.isEmpty) ? 'Campo obligatorio' : null
                            : null,
                        onChanged:       (_) => ref.read(authProvider.notifier).clearError(),
                      ),
                      const SizedBox(height: 24),

                      // Botón
                      AuthButton(
                        label:     'Iniciar sesión',
                        onPressed: _submit,
                        isLoading: isLoading,
                      ),
                    ],
                  ),
                ),
              ),

              const SizedBox(height: 24),

              // Link al registro
              Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text('¿No tienes cuenta? ', style: tt.bodyMedium),
                  TextButton(
                    onPressed: () => context.push('/register'),
                    child: const Text('Regístrate'),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## 3.6 Pantalla Registro — `presentation/screens/auth/register_screen.dart`

```dart
// lib/presentation/screens/auth/register_screen.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../../theme/app_colors.dart';
import '../../../core/utils/validators.dart';
import '../../../domain/model/auth_state.dart';
import '../../providers/auth_provider.dart';
import '../../widgets/auth_button.dart';
import '../../widgets/auth_text_field.dart';

class RegisterScreen extends ConsumerStatefulWidget {
  const RegisterScreen({super.key});

  @override
  ConsumerState<RegisterScreen> createState() => _RegisterScreenState();
}

class _RegisterScreenState extends ConsumerState<RegisterScreen> {
  final _formKey     = GlobalKey<FormState>();
  final _userCtrl    = TextEditingController();
  final _emailCtrl   = TextEditingController();
  final _passCtrl    = TextEditingController();
  final _pass2Ctrl   = TextEditingController();
  bool  _submitted   = false;

  @override
  void dispose() {
    _userCtrl.dispose();
    _emailCtrl.dispose();
    _passCtrl.dispose();
    _pass2Ctrl.dispose();
    super.dispose();
  }

  Future<void> _submit() async {
    setState(() => _submitted = true);
    if (!_formKey.currentState!.validate()) return;
    await ref.read(authProvider.notifier).register(
      _userCtrl.text,
      _emailCtrl.text,
      _passCtrl.text,
      _pass2Ctrl.text,
    );
  }

  @override
  Widget build(BuildContext context) {
    final authState = ref.watch(authProvider);
    final isLoading = authState.isChecking;
    final error     = authState.error;
    final tt        = Theme.of(context).textTheme;

    ref.listen<AuthState>(authProvider, (_, next) {
      if (next.isAuthenticated) {
        context.go(next.isStaff ? '/admin' : '/');
      }
    });

    return Scaffold(
      body: SafeArea(
        child: SingleChildScrollView(
          padding: const EdgeInsets.symmetric(horizontal: 24),
          child: Column(
            children: [
              const SizedBox(height: 60),
              Text('Flutter Shop App', style: tt.displayMedium?.copyWith(color: AppColors.accent)),
              const SizedBox(height: 8),
              Text('Crea tu cuenta gratis', style: tt.bodyMedium),
              const SizedBox(height: 40),

              Container(
                padding:    const EdgeInsets.all(24),
                decoration: BoxDecoration(
                  color:        AppColors.surface,
                  borderRadius: BorderRadius.circular(20),
                  border:       Border.all(color: AppColors.border),
                ),
                child: Form(
                  key: _formKey,
                  child: Column(
                    children: [

                      if (error != null) ...[
                        Container(
                          width:   double.infinity,
                          padding: const EdgeInsets.all(12),
                          decoration: BoxDecoration(
                            color:        AppColors.error.withValues(alpha: 0.1),
                            borderRadius: BorderRadius.circular(10),
                          ),
                          child: Text(
                            error,
                            style: const TextStyle(color: AppColors.error, fontSize: 13),
                          ),
                        ),
                        const SizedBox(height: 16),
                      ],

                      // Usuario
                      AuthTextField(
                        label:      'Usuario',
                        hint:       'mínimo 3 caracteres',
                        controller: _userCtrl,
                        enabled:    !isLoading,
                        validator:  _submitted ? validateUsername : null,
                        onChanged:  (_) => ref.read(authProvider.notifier).clearError(),
                      ),
                      const SizedBox(height: 14),

                      // Email
                      AuthTextField(
                        label:       'Email',
                        hint:        'tu@email.com',
                        controller:  _emailCtrl,
                        enabled:     !isLoading,
                        keyboardType:TextInputType.emailAddress,
                        validator:   _submitted ? validateEmail : null,
                        onChanged:   (_) => ref.read(authProvider.notifier).clearError(),
                      ),
                      const SizedBox(height: 14),

                      // Contraseña
                      AuthTextField(
                        label:      'Contraseña',
                        hint:       'mínimo 8 caracteres',
                        controller: _passCtrl,
                        isPassword: true,
                        enabled:    !isLoading,
                        validator:  _submitted ? validatePassword : null,
                        onChanged:  (_) => ref.read(authProvider.notifier).clearError(),
                      ),
                      const SizedBox(height: 14),

                      // Confirmar contraseña
                      AuthTextField(
                        label:           'Confirmar contraseña',
                        hint:            'repite la contraseña',
                        controller:      _pass2Ctrl,
                        isPassword:      true,
                        enabled:         !isLoading,
                        textInputAction: TextInputAction.done,
                        validator:       _submitted
                            ? (v) {
                                if (v != _passCtrl.text) return 'Las contraseñas no coinciden';
                                return null;
                              }
                            : null,
                        onChanged: (_) => ref.read(authProvider.notifier).clearError(),
                      ),
                      const SizedBox(height: 24),

                      AuthButton(
                        label:     'Crear mi cuenta',
                        onPressed: _submit,
                        isLoading: isLoading,
                      ),
                    ],
                  ),
                ),
              ),

              const SizedBox(height: 24),
              Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text('¿Ya tienes cuenta? ', style: tt.bodyMedium),
                  TextButton(
                    onPressed: () => context.go('/login'),
                    child: const Text('Inicia sesión'),
                  ),
                ],
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## 3.7 Router provisional — `presentation/navigation/app_router.dart`

```dart
// lib/presentation/navigation/app_router.dart

import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:go_router/go_router.dart';
import '../../domain/model/auth_state.dart';
import '../providers/auth_provider.dart';
import '../screens/auth/login_screen.dart';
import '../screens/auth/register_screen.dart';

// Pantalla temporal para los placeholders
class _SplashScreen extends StatelessWidget {
  const _SplashScreen();

  @override
  Widget build(BuildContext context) => const Scaffold(
    body: Center(child: CircularProgressIndicator()),
  );
}

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
      body: Center(child: Text(title, style: const TextStyle(fontSize: 18))),
    );
  }
}

final routerProvider = Provider<GoRouter>((ref) {
  return GoRouter(
    initialLocation: '/splash',
    refreshListenable: _AuthStateListenable(ref),
    redirect: (context, state) {
      final authState = ref.read(authProvider);
      final isChecking = authState.isChecking;
      final isAuth     = authState.isAuthenticated;
      final isStaff    = authState.isStaff;
      final location   = state.matchedLocation;

      // Mientras se verifica la sesión, mostrar splash.
      if (isChecking) {
        return location == '/splash' ? null : '/splash';
      }

      final isAuthRoute = location == '/login' || location == '/register';
      final isSplash    = location == '/splash';

      // Al terminar la verificación, salir del splash.
      if (isSplash) return isAuth ? (isStaff ? '/admin' : '/') : '/login';

      // No autenticado → solo puede ir a auth
      if (!isAuth && !isAuthRoute) return '/login';

      // Autenticado → no puede ir a auth
      if (isAuth && isAuthRoute) return isStaff ? '/admin' : '/';

      // Cliente intenta acceder a admin → redirigir a home
      if (isAuth && !isStaff && location.startsWith('/admin')) return '/';

      return null;
    },
    routes: [
      GoRoute(
        path:    '/splash',
        builder: (_, __) => const _SplashScreen(),
      ),

      // ── Auth ──────────────────────────────────────────────
      GoRoute(
        path:    '/login',
        builder: (_, __) => const LoginScreen(),
      ),
      GoRoute(
        path:    '/register',
        builder: (_, __) => const RegisterScreen(),
      ),

      // ── Público ───────────────────────────────────────────
      GoRoute(
        path:    '/',
        builder: (_, __) => const _PlaceholderScreen('Home — M5'),
      ),
      GoRoute(
        path:    '/catalog',
        builder: (_, __) => const _PlaceholderScreen('Catálogo — M5'),
      ),
      GoRoute(
        path:    '/product/:id',
        builder: (_, __) => const _PlaceholderScreen('Detalle — M5'),
      ),

      // ── Cliente privado ───────────────────────────────────
      GoRoute(
        path:    '/orders',
        builder: (_, __) => const _PlaceholderScreen('Mis pedidos — M7'),
      ),
      GoRoute(
        path:    '/profile',
        builder: (_, __) => const _PlaceholderScreen('Perfil — M7'),
      ),

      // ── Admin ─────────────────────────────────────────────
      GoRoute(
        path:    '/admin',
        builder: (_, __) => const _PlaceholderScreen('Dashboard — M8'),
      ),
    ],
  );
});

// Listenable que notifica al router cuando cambia el AuthState
class _AuthStateListenable extends ChangeNotifier {
  _AuthStateListenable(Ref ref) {
    ref.listen<AuthState>(authProvider, (_, __) => notifyListeners());
  }
}
```

---

## 3.8 Actualizar `main.dart` — integrar router

```dart
// lib/main.dart

import 'package:flutter/material.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'presentation/navigation/app_router.dart';
import 'theme/app_theme.dart';

Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await dotenv.load(fileName: '.env');
  runApp(const ProviderScope(child: FlutterShopApp()));
}

class FlutterShopApp extends ConsumerWidget {
  const FlutterShopApp({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final router = ref.watch(routerProvider);
    return MaterialApp.router(
      title:                'Flutter Shop App',
      debugShowCheckedModeBanner: false,
      theme:                AppTheme.dark,
      routerConfig:         router,
    );
  }
}
```

---

## ✅ Checkpoint Módulo 3

### Crear usuarios de prueba en Django

```bash
uv run python manage.py createsuperuser
# username: admin | password: Admin1234!

uv run python manage.py shell -c "
from django.contrib.auth.models import User
User.objects.create_user('cliente', 'cliente@test.com', 'Pass1234!')
print('Usuario cliente creado')
"
```

### Flujo a verificar en el emulador

### Verificación local

```bash
flutter analyze
flutter test
```

| # | Acción | Resultado esperado |
|---|--------|--------------------|
| 1 | Arrancar la app | Splash breve → pantalla Login |
| 2 | Login con campos vacíos → Submit | Errores de validación rojos |
| 3 | Login con credenciales incorrectas | Banner rojo con mensaje de Django |
| 4 | Login con cuenta `cliente` | Navega a `Home — M5` (placeholder) |
| 5 | Cerrar app y reabrir | Sesión restaurada, va directo a Home |
| 6 | Login con cuenta `admin` (staff) | Navega a `Dashboard — M8` (placeholder) |
| 7 | Ir a `/register` | Pantalla de registro |
| 8 | Contraseñas distintas | Error "Las contraseñas no coinciden" |
| 9 | Registro válido | Navega a Home |
| 10 | Usuario autenticado → intenta `/login` | GoRouter redirige a Home automáticamente |

| Comando | Resultado esperado |
|---|---|
| `flutter analyze` | `No issues found!` |
| `flutter test` | `All tests passed!` |

---

## Resumen

| Elemento | Estado |
|---|---|
| `AuthRemoteDatasource` — login, register, logout | ✅ |
| `AuthState` con enum `AuthStatus` | ✅ |
| `AuthNotifier` con restauración de sesión en `init` | ✅ |
| `ref.listen` para navegar tras login/registro | ✅ |
| `LoginScreen` con `Form` y `GlobalKey` | ✅ |
| `RegisterScreen` con validación de contraseñas | ✅ |
| `AuthTextField` con toggle de contraseña | ✅ |
| `AuthButton` con estado loading | ✅ |
| GoRouter con `redirect` y `refreshListenable` | ✅ |
| Guard de roles: cliente → Home, staff → Admin | ✅ |
| `_AuthStateListenable` que notifica al router | ✅ |

**Siguiente módulo →** M4: GoRouter completo — ShellRoute, BottomNavBar y navegación por zonas
