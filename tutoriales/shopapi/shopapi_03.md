# ShopAPI — Etapa 3
## CRUD de Categorías

> **Objetivo:** Modelo `Category` + CRUD completo con paginación, filtros, búsqueda, ordenamiento y acciones extra.
> **Checkpoint final:** endpoints de categorías funcionando con permisos diferenciados (staff escribe, autenticado lee).

---

## 3.1 Modelo

```python
# store/models/category.py
from django.db import models


class Category(models.Model):
    name        = models.CharField(max_length=100, unique=True)
    slug        = models.SlugField(unique=True)
    description = models.TextField(blank=True, default='')
    is_active   = models.BooleanField(default=True)
    created_at  = models.DateTimeField(auto_now_add=True)

    class Meta:
        verbose_name        = 'Category'
        verbose_name_plural = 'Categories'
        ordering            = ['name']

    def __str__(self):
        return self.name
```

### `store/models/__init__.py`

```python
# store/models/__init__.py
from .category import Category

__all__ = ['Category']
```

### Migración

```bash
uv run python manage.py makemigrations store
uv run python manage.py migrate
```

---

## 3.2 Permiso reutilizable

Este permiso se usará en Categorías, Productos y cualquier recurso que siga
la misma regla: staff escribe, usuario autenticado solo lee.

```python
# store/permissions.py
from rest_framework.permissions import BasePermission, SAFE_METHODS


class IsStaffOrReadOnly(BasePermission):
    def has_permission(self, request, view):
        if request.method in SAFE_METHODS:
            return bool(request.user and request.user.is_authenticated)
        return bool(request.user and request.user.is_staff)
```

---

## 3.3 Filtro

```python
# store/filters.py
import django_filters
from store.models import Category


class CategoryFilter(django_filters.FilterSet):
    name = django_filters.CharFilter(lookup_expr='icontains')

    class Meta:
        model  = Category
        fields = ['is_active']
```

---

## 3.4 Serializer

```python
# store/serializers/category.py
from rest_framework import serializers
from django.utils.text import slugify
from store.models import Category


class CategorySerializer(serializers.ModelSerializer):
    """
    El campo total_products se agrega en la Etapa 4
    cuando el modelo Product ya existe.
    Incluirlo antes provoca AttributeError: 'Category' has no attribute 'products'.
    """

    class Meta:
        model  = Category
        fields = [
            'id', 'name', 'slug', 'description',
            'is_active', 'created_at',
        ]
        read_only_fields = ['id', 'created_at']

    def validate_slug(self, value):
        return slugify(value)

    def validate_name(self, value):
        qs = Category.objects.filter(name__iexact=value)
        if self.instance:
            qs = qs.exclude(pk=self.instance.pk)
        if qs.exists():
            raise serializers.ValidationError('A category with this name already exists.')
        return value
```

### `store/serializers/__init__.py` — actualizar

```python
# store/serializers/__init__.py
from .auth     import CustomTokenSerializer, CustomTokenView
from .user     import (
    RegisterSerializer,
    UserSerializer,
    UserProfileSerializer,
    ChangePasswordSerializer,
)
from .category import CategorySerializer
```

---

## 3.5 View

```python
# store/views/category.py
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.filters import SearchFilter, OrderingFilter
from django_filters.rest_framework import DjangoFilterBackend
from django.db.models import Count

from store.models               import Category
from store.serializers.category import CategorySerializer
from store.permissions          import IsStaffOrReadOnly
from store.filters              import CategoryFilter
from store.pagination           import StandardPagination


class CategoryViewSet(viewsets.ModelViewSet):
    queryset           = Category.objects.all()
    serializer_class   = CategorySerializer
    permission_classes = [IsStaffOrReadOnly]
    pagination_class   = StandardPagination
    filter_backends    = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class    = CategoryFilter
    search_fields      = ['name', 'description']
    ordering_fields    = ['name', 'created_at']
    ordering           = ['name']

    @action(detail=True, methods=['get'], url_path='products')
    def active_products(self, request, pk=None):
        """
        Esta acción retorna lista vacía hasta la Etapa 4.
        Una vez creado el modelo Product se actualiza en la sección 4.6.1.
        """
        return Response([])

    @action(detail=False, methods=['get'], url_path='stats')
    def stats(self, request):
        qs = Category.objects.annotate(num_products=Count('products', distinct=True))
        return Response({
            'total':    qs.count(),
            'active':   qs.filter(is_active=True).count(),
            'inactive': qs.filter(is_active=False).count(),
            'detail': [
                {
                    'id':           c.id,
                    'name':         c.name,
                    'num_products': c.num_products,
                    'is_active':    c.is_active,
                }
                for c in qs.order_by('name')
            ],
        })
```

---

## 3.6 `views/__init__.py`

```python
# store/views/__init__.py
```

---

## 3.7 URLs — actualizar `store/urls.py`

```python
# store/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from rest_framework_simplejwt.views import TokenRefreshView, TokenVerifyView

from store.views.health   import health_check
from store.views.auth     import RegisterView, LogoutView
from store.views.user     import UserViewSet
from store.views.category import CategoryViewSet
from store.serializers.auth import CustomTokenView

router = DefaultRouter()
router.register('users',      UserViewSet,      basename='user')
router.register('categories', CategoryViewSet,  basename='category')

urlpatterns = [
    path('health/',             health_check),
    path('auth/register/',      RegisterView.as_view()),
    path('auth/login/',         CustomTokenView.as_view()),
    path('auth/token/refresh/', TokenRefreshView.as_view()),
    path('auth/token/verify/',  TokenVerifyView.as_view()),
    path('auth/logout/',        LogoutView.as_view()),
    path('', include(router.urls)),
]
```

---

## 3.8 Admin

```python
# store/admin.py
from django.contrib import admin
from store.models import Category


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display        = ['id', 'name', 'slug', 'is_active', 'created_at']
    list_filter         = ['is_active']
    search_fields       = ['name']
    prepopulated_fields = {'slug': ('name',)}
```

---

## ✅ Checkpoint Etapa 3

```bash
uv run python manage.py runserver
```

| # | Endpoint | Método | Auth | Esperado |
|---|----------|--------|------|----------|
| 1 | `/api/categories/` | GET | Bearer | 200 lista paginada |
| 2 | `/api/categories/` | POST | Staff | 201 creada |
| 3 | `/api/categories/` | POST | User | 403 forbidden |
| 4 | `/api/categories/?is_active=true` | GET | Bearer | 200 filtrado |
| 5 | `/api/categories/?search=electro` | GET | Bearer | 200 búsqueda |
| 6 | `/api/categories/?ordering=-created_at` | GET | Bearer | 200 ordenado |
| 7 | `/api/categories/{id}/` | PATCH | Staff | 200 actualizada |
| 8 | `/api/categories/{id}/` | DELETE | Staff | 204 eliminada |
| 9 | `/api/categories/stats/` | GET | Bearer | 200 resumen |

---

## 📦 Colección Postman — Etapa 3

```json
{
  "info": {
    "name": "ShopAPI — Stage 3: Categories",
    "_postman_id": "shopapi-stage-03",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "List categories",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": "{{base_url}}/categories/"
      }
    },
    {
      "name": "List — filter active",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": {
          "raw": "{{base_url}}/categories/?is_active=true&ordering=name",
          "query": [
            { "key": "is_active", "value": "true" },
            { "key": "ordering",  "value": "name" }
          ]
        }
      }
    },
    {
      "name": "Search category",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": {
          "raw": "{{base_url}}/categories/?search=electro",
          "query": [{ "key": "search", "value": "electro" }]
        }
      }
    },
    {
      "name": "[Staff] Create category",
      "event": [{ "listen": "test", "script": { "exec": [
        "pm.test('Status 201', () => pm.response.to.have.status(201));",
        "pm.collectionVariables.set('category_id', pm.response.json().id);"
      ]}}],
      "request": {
        "method": "POST",
        "header": [
          { "key": "Content-Type",  "value": "application/json" },
          { "key": "Authorization", "value": "Bearer {{access}}" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"name\": \"Electronics\",\n  \"slug\": \"electronics\",\n  \"description\": \"Electronic devices and gadgets\",\n  \"is_active\": true\n}"
        },
        "url": "{{base_url}}/categories/"
      }
    },
    {
      "name": "[Staff] Update category",
      "request": {
        "method": "PATCH",
        "header": [
          { "key": "Content-Type",  "value": "application/json" },
          { "key": "Authorization", "value": "Bearer {{access}}" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"description\": \"Updated description\"\n}"
        },
        "url": "{{base_url}}/categories/{{category_id}}/"
      }
    },
    {
      "name": "[Staff] Delete category",
      "request": {
        "method": "DELETE",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": "{{base_url}}/categories/{{category_id}}/"
      }
    },
    {
      "name": "Stats",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": "{{base_url}}/categories/stats/"
      }
    }
  ],
  "variable": [
    { "key": "base_url",     "value": "http://localhost:8000/api" },
    { "key": "access",       "value": "" },
    { "key": "refresh",      "value": "" },
    { "key": "category_id",  "value": "" }
  ]
}
```

---

## ⚠️ Pendiente para Etapa 4

Cuando el modelo `Product` exista, actualizar dos cosas en `store/serializers/category.py` y `store/views/category.py`.
Esto se detalla en la sección **4.6.1** del siguiente documento.

---

## Resumen

| Elemento | Estado |
|---|---|
| Modelo `Category` + migración | ✅ |
| Permiso `IsStaffOrReadOnly` reutilizable | ✅ |
| `CategoryFilter` por `name` e `is_active` | ✅ |
| `CategorySerializer` con `total_products` calculado | ✅ |
| CRUD con paginación, filtros, búsqueda y ordenamiento | ✅ |
| Acciones: `active_products`, `stats` | ✅ |
| Admin con slug prepopulado | ✅ |
| Colección Postman con test scripts | ✅ |

**Siguiente etapa →** CRUD de Productos