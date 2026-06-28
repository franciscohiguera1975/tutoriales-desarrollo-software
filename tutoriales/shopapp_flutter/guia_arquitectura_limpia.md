# Guía de Arquitectura Limpia — Flutter Shop App

> Documento de estudio independiente del tutorial.
> Explica el **por qué** detrás de cada decisión de estructura.

---

## ¿Qué es la Arquitectura Limpia?

La Arquitectura Limpia (Clean Architecture) es un conjunto de principios que organiza el código en **capas concéntricas**. La regla fundamental es:

> **Las capas internas no conocen a las capas externas.**
> Las dependencias siempre apuntan hacia adentro.

```
┌─────────────────────────────────────┐
│         PRESENTATION                │  ← Ve pantallas, widgets, providers
│  ┌───────────────────────────────┐  │
│  │          DATA                 │  │  ← Habla con APIs, base de datos
│  │  ┌─────────────────────────┐  │  │
│  │  │        DOMAIN           │  │  │  ← Reglas de negocio puras
│  │  │  (modelos + contratos)  │  │  │
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
         CORE  ← utilidades transversales (no es una capa, es soporte)
```

---

## Las 3 capas + core

### 1. `domain/` — El corazón de la app

Es la capa **más interna**. No depende de Flutter, Dio, ni ningún paquete externo.
Contiene únicamente:

| Carpeta | Qué va | Ejemplo |
|---|---|---|
| `domain/model/` | Clases de datos puras del negocio | `Product`, `Order`, `User` |
| `domain/repository/` | **Contratos** (interfaces abstractas) | `abstract class CategoryRepository` |

**Regla de oro:** si borras Flutter del proyecto, `domain/` debe seguir compilando.

```dart
// domain/model/product.dart
// ✅ Solo Dart puro — sin imports de Flutter ni de paquetes
class Product {
  final int    id;
  final String name;
  final double price;
  const Product({required this.id, required this.name, required this.price});
}
```

```dart
// domain/repository/category_repository.dart
// ✅ Contrato abstracto — define QUÉ puede hacer, no CÓMO
import '../model/category.dart';

abstract class CategoryRepository {
  Future<List<Category>> getCategories();
  Future<Category>       getCategory(int id);
}
```

---

### 2. `data/` — La capa de datos

Implementa los contratos definidos en `domain/`. Aquí sí se usan paquetes externos como Dio y FlutterSecureStorage.

| Carpeta | Qué va | Ejemplo |
|---|---|---|
| `data/remote/api/` | Cliente HTTP y datasources remotos | `dio_client.dart`, `category_remote_datasource.dart` |
| `data/remote/dto/` | Objetos de transferencia de datos (JSON → modelo) | `category_dto.dart` |
| `data/remote/interceptor/` | Interceptores Dio adicionales | `auth_interceptor.dart` |
| `data/local/` | Almacenamiento local | `secure_storage.dart` |
| `data/repository/` | Implementaciones de los contratos de `domain/` | `CategoryRepositoryImpl` |

```dart
// data/repository/category_repository_impl.dart
// ✅ Implementa el contrato del dominio
import '../../domain/repository/category_repository.dart';  // contrato
import '../../domain/model/category.dart';                  // modelo
import '../remote/api/category_remote_datasource.dart';     // fuente de datos

class CategoryRepositoryImpl implements CategoryRepository {
  final CategoryRemoteDatasource _datasource;
  CategoryRepositoryImpl(this._datasource);

  @override
  Future<List<Category>> getCategories() => _datasource.getCategories();
}
```

**¿Por qué separar datasource de repository?**

| Capa | Responsabilidad |
|---|---|
| `datasource` | Habla con la API. Convierte JSON → modelo. Lanza `ApiException`. |
| `repository` | Orquesta fuentes de datos (remota + local). Decide cuándo usar caché. |

---

### 3. `presentation/` — La capa de UI

Contiene todo lo que el usuario ve e interactúa. Usa `domain/` (modelos y contratos) pero **nunca** importa de `data/` directamente.

| Carpeta | Qué va | Ejemplo |
|---|---|---|
| `presentation/screens/` | Pantallas completas | `login_screen.dart` |
| `presentation/widgets/` | Componentes reutilizables | `product_card.dart` |
| `presentation/providers/` | Estado (Riverpod) | `auth_provider.dart` |
| `presentation/navigation/` | Rutas (GoRouter) | `app_router.dart` |

```dart
// presentation/providers/auth_provider.dart
// ✅ Solo usa el contrato del dominio, no la implementación
import '../../domain/repository/auth_repository.dart';  // contrato
import '../../domain/model/auth_models.dart';           // modelo
import '../../core/error/api_exception.dart';           // error compartido
```

---

### 4. `core/` — Soporte transversal

**No es una capa de la arquitectura.** Es un conjunto de utilidades y clases que necesitan TODAS las capas.

| Carpeta | Qué va | Ejemplo |
|---|---|---|
| `core/config/` | Configuración global | `AppConfig` (URL base, nombre app) |
| `core/error/` | Excepciones compartidas | `ApiException` |
| `core/utils/` | Funciones utilitarias | `formatters.dart`, `validators.dart` |

**¿Por qué `ApiException` está en `core/error/` y no en `data/remote/api/`?**

`ApiException` es lanzada en la capa `data` pero capturada en la capa `presentation`. Si estuviera en `data/remote/api/`, la capa `presentation` tendría que importar de `data/`, rompiendo la regla de dependencias.

Al ponerla en `core/`, cualquier capa puede importarla sin crear dependencias incorrectas.

---

## Estructura de carpetas completa

```
lib/
├── main.dart
│
├── core/                               ← soporte transversal
│   ├── config/
│   │   └── app_config.dart             ← URL, nombre app, constantes
│   ├── error/
│   │   └── api_exception.dart          ← excepción HTTP compartida
│   └── utils/
│       ├── formatters.dart             ← formato de precios, fechas
│       └── validators.dart             ← validaciones de formularios
│
├── data/                               ← capa de datos
│   ├── remote/
│   │   ├── api/
│   │   │   └── dio_client.dart         ← cliente HTTP + interceptor JWT
│   │   ├── dto/                        ← objetos JSON (si se usan)
│   │   └── interceptor/                ← interceptores adicionales
│   ├── local/
│   │   └── secure_storage.dart         ← tokens en almacenamiento seguro
│   └── repository/                     ← implementaciones de contratos
│       ├── auth_repository_impl.dart
│       ├── catalog_repository_impl.dart
│       ├── order_repository_impl.dart
│       └── admin_repository_impl.dart
│
├── domain/                             ← capa de dominio (núcleo)
│   ├── model/                          ← entidades del negocio
│   │   ├── auth_models.dart
│   │   ├── category.dart
│   │   ├── product.dart
│   │   ├── order.dart
│   │   └── user.dart
│   └── repository/                     ← contratos abstractos
│       ├── auth_repository.dart
│       ├── catalog_repository.dart
│       ├── order_repository.dart
│       └── admin_repository.dart
│
├── presentation/                       ← capa de UI
│   ├── navigation/
│   │   └── app_router.dart
│   ├── screens/
│   │   ├── auth/
│   │   ├── catalog/
│   │   ├── cart/
│   │   ├── orders/
│   │   ├── profile/
│   │   └── admin/
│   ├── providers/                      ← estado global (Riverpod)
│   └── widgets/                        ← componentes reutilizables
│
└── theme/                              ← design system
    ├── app_colors.dart
    ├── app_text_styles.dart
    └── app_theme.dart
```

---

## Regla de dependencias visualizada

```
presentation  →  domain   ✅
presentation  →  core     ✅
presentation  →  data     ❌  (nunca directo)

data          →  domain   ✅
data          →  core     ✅
data          →  presentation ❌

domain        →  nada     ✅  (solo Dart puro)
domain        →  data     ❌
domain        →  presentation ❌

core          →  nada     ✅  (utilidades independientes)
```

---

## Tabla de imports por ubicación

Cada `..` en una ruta sube UN nivel de directorio.

### Desde `data/remote/api/`
```
lib/data/remote/api/archivo.dart

../          = lib/data/remote/
../../       = lib/data/
../../../    = lib/

'../../../core/error/api_exception.dart'    → lib/core/error/
'../../../core/config/app_config.dart'      → lib/core/config/
'../../../domain/model/xxx.dart'            → lib/domain/model/
'../../local/secure_storage.dart'           → lib/data/local/
```

### Desde `data/repository/`
```
lib/data/repository/archivo.dart

../          = lib/data/
../../       = lib/

'../../core/error/api_exception.dart'       → lib/core/error/
'../../domain/model/xxx.dart'               → lib/domain/model/
'../../domain/repository/xxx.dart'          → lib/domain/repository/
'../remote/api/xxx_datasource.dart'         → lib/data/remote/api/
```

### Desde `presentation/providers/`
```
lib/presentation/providers/archivo.dart

../          = lib/presentation/
../../       = lib/

'../../core/error/api_exception.dart'       → lib/core/error/
'../../domain/model/xxx.dart'               → lib/domain/model/
'../../domain/repository/xxx.dart'          → lib/domain/repository/
```

### Desde `presentation/screens/auth/`
```
lib/presentation/screens/auth/archivo.dart

../          = lib/presentation/screens/
../../       = lib/presentation/
../../../    = lib/

'../../../core/error/api_exception.dart'    → lib/core/error/
'../../../domain/model/xxx.dart'            → lib/domain/model/
'../widgets/xxx.dart'                       → lib/presentation/screens/widgets/ ❌
'../../widgets/xxx.dart'                    → lib/presentation/widgets/ ✅
```

---

## Flujo de datos completo — ejemplo: cargar categorías

```
[UI: CatalogScreen]
       │
       │ ref.watch(catalogProvider)
       ▼
[Provider: catalog_provider.dart]         ← presentation/providers/
       │
       │ categoryRepository.getCategories()
       │ (usa el contrato abstracto, no la implementación)
       ▼
[Contrato: CategoryRepository]            ← domain/repository/
       │
       │ (Riverpod inyecta CategoryRepositoryImpl)
       ▼
[Impl: CategoryRepositoryImpl]            ← data/repository/
       │
       │ _datasource.getCategories()
       ▼
[Datasource: CategoryRemoteDatasource]    ← data/remote/api/
       │
       │ _dio.get('/categories/')
       ▼
[DioClient]                               ← data/remote/api/
       │
       │ HTTP GET https://api.site/categories/
       ▼
[Django REST API]
       │
       │ JSON response
       ▼
[Category.fromJson()]                     ← domain/model/
       │
       ▼
[UI recibe List<Category>]
```

---

## Beneficios en la práctica

| Beneficio | Explicación |
|---|---|
| **Testeable** | Se puede probar `domain/` sin levantar un servidor |
| **Intercambiable** | Cambiar de Dio a http solo afecta `data/remote/api/` |
| **Escalable** | Agregar una feature nueva sigue siempre el mismo patrón |
| **Legible** | Cualquier desarrollador sabe dónde buscar cada cosa |
| **Sin dependencias circulares** | La dirección de imports es siempre hacia adentro |

---

## Evolución: Estructura por Features (Feature-Based Architecture)

Cuando el proyecto crece, la estructura por capas puede volverse difícil de mantener. La solución es **organizar por features** manteniendo los principios de Clean Architecture.

### ¿Cuándo cambiar a estructura por features?

- **Proyectos pequeños (< 10 features):** Mantener estructura por capas es suficiente
- **Proyectos medianos/grandes (10+ features):** Estructura por features mejora organización

### Estructura por features

```
lib/
├── main.dart
│
├── core/                               ← soporte transversal (igual que antes)
│   ├── config/
│   ├── error/
│   └── utils/
│
├── features/                           ← cada feature es un módulo autónomo
│   ├── auth/                           ← feature de autenticación
│   │   ├── data/
│   │   │   ├── remote/
│   │   │   │   └── api/
│   │   │   │       └── auth_remote_datasource.dart
│   │   │   └── repository/
│   │   │       └── auth_repository_impl.dart
│   │   ├── domain/
│   │   │   ├── model/
│   │   │   │   └── auth_models.dart
│   │   │   └── repository/
│   │   │       └── auth_repository.dart
│   │   └── presentation/
│   │       ├── providers/
│   │       │   └── auth_provider.dart
│   │       └── screens/
│   │           ├── login_screen.dart
│   │           └── register_screen.dart
│   │
│   ├── catalog/                        ← feature de catálogo
│   │   ├── data/
│   │   │   ├── remote/
│   │   │   │   └── api/
│   │   │   │       ├── category_remote_datasource.dart
│   │   │   │       └── product_remote_datasource.dart
│   │   │   └── repository/
│   │   │       ├── category_repository_impl.dart
│   │   │       └── product_repository_impl.dart
│   │   ├── domain/
│   │   │   ├── model/
│   │   │   │   ├── category.dart
│   │   │   │   └── product.dart
│   │   │   └── repository/
│   │   │       ├── category_repository.dart
│   │   │       └── product_repository.dart
│   │   └── presentation/
│   │       ├── providers/
│   │       │   └── catalog_provider.dart
│   │       └── screens/
│   │           ├── catalog_screen.dart
│   │           └── product_detail_screen.dart
│   │
│   ├── orders/                         ← feature de pedidos
│   │   ├── data/
│   │   ├── domain/
│   │   └── presentation/
│   │
│   └── admin/                          ← feature de administración
│       ├── data/
│       ├── domain/
│       └── presentation/
│
├── presentation/                       ← elementos compartidos de UI
│   ├── navigation/
│   │   └── app_router.dart
│   └── widgets/
│       ├── product_card.dart
│       └── status_badge.dart
│
└── theme/                              ← design system (igual que antes)
    ├── app_colors.dart
    ├── app_text_styles.dart
    └── app_theme.dart
```

### Imports en estructura por features

Desde `features/catalog/presentation/providers/catalog_provider.dart`:

```dart
// ✅ Importar dentro del mismo feature
import '../../data/remote/api/product_remote_datasource.dart';
import '../../domain/model/product.dart';
import '../../domain/repository/product_repository.dart';

// ✅ Importar de core (siempre desde la raíz de lib/)
import '../../../core/error/api_exception.dart';
import '../../../core/utils/formatters.dart';

// ✅ Importar de presentation compartido
import '../../../presentation/widgets/product_card.dart';

// ❌ NUNCA importar de otro feature directamente
// import '../../auth/presentation/providers/auth_provider.dart';
```

### Reglas de dependencias en estructura por features

```
feature/catalog/presentation  →  feature/catalog/domain     ✅
feature/catalog/presentation  →  feature/catalog/data      ✅
feature/catalog/presentation  →  core/                      ✅
feature/catalog/presentation  →  presentation/ (compartido) ✅
feature/catalog/presentation  →  feature/auth/              ❌ (usar core/ o eventos)
```

### Comunicación entre features

Cuando un feature necesita datos de otro, usa:

1. **Core compartido:** Modelos, excepciones, utilidades
2. **Eventos globales:** Riverpod providers en `core/` o `presentation/providers/`
3. **Contratos en domain:** Interfaces que cualquier feature puede implementar

### Ventajas de la estructura por features

| Ventaja | Explicación |
|---|---|
| **Aislamiento** | Cada feature es autónomo, fácil de mover/reutilizar |
| **Colaboración** | Equipos pueden trabajar en features sin conflictos |
| **Onboarding** | Nuevo dev solo estudia el feature relevante |
| **Delete-friendly** | Borrar un feature es eliminar una carpeta |
| **Testing** | Tests por feature son más enfocados |

### Migración de capas a features

No es una reescritura completa. Migrar gradualmente:

1. Crear carpeta `features/`
2. Mover un feature completo (ej: `auth/`)
3. Actualizar imports
4. Verificar que todo compila
5. Repetir con siguiente feature
6. Al final, eliminar carpetas vacías de `data/`, `domain/`, `presentation/`

### Cuál estructura elegir

| Criterio | Por capas | Por features |
|---|---|---|
| Tamaño proyecto | < 10 features | 10+ features |
| Equipo | 1-2 devs | 3+ devs |
| Complejidad | Baja | Media/Alta |
| Reutilización | Poca | Alta (features como módulos) |

---

## Resumen final

La Clean Architecture no es una estructura de carpetas fija. Son **principios**:

1. **Dependencias hacia adentro:** Las capas internas no conocen las externas
2. **Dominio puro:** `domain/` sin dependencias de frameworks
3. **Implementación en data:** `data/` implementa contratos de `domain/`
4. **UI agnóstica:** `presentation/` solo conoce contratos, no implementaciones

La estructura por capas es el punto de partida. La estructura por features es la evolución natural cuando el proyecto escala. Ambas respetan los mismos principios.
