# ShopAPI — Etapa 4
## CRUD de Productos

> **Objetivo:** Modelo `Product` con FK a `Category` + CRUD completo con campos calculados, filtros de rango, paginación y acciones extra.
> **Checkpoint final:** endpoints de productos funcionando con filtros por precio, stock y categoría, y acciones `restock`, `available` y `stats`.

---

## 4.1 Modelo

```python
# store/models/product.py
from django.db import models
from .category import Category


class Product(models.Model):
    name        = models.CharField(max_length=200)
    description = models.TextField(blank=True, default='')
    price       = models.DecimalField(max_digits=10, decimal_places=2)
    stock       = models.PositiveIntegerField(default=0)
    is_active   = models.BooleanField(default=True)
    category    = models.ForeignKey(
        Category,
        on_delete=models.PROTECT,
        related_name='products',
    )
    created_at  = models.DateTimeField(auto_now_add=True)
    updated_at  = models.DateTimeField(auto_now=True)

    class Meta:
        ordering = ['name']

    def __str__(self):
        return self.name

    @property
    def price_with_tax(self):
        return round(float(self.price) * 1.15, 2)

    @property
    def in_stock(self):
        return self.stock > 0
```

### `store/models/__init__.py` — actualizar

```python
# store/models/__init__.py
from .category import Category
from .product  import Product

__all__ = ['Category', 'Product']
```

### Migración

```bash
uv run python manage.py makemigrations store
uv run python manage.py migrate
```

---

## 4.2 Filtro — actualizar `store/filters.py`

```python
# store/filters.py
import django_filters
from store.models import Category, Product


class CategoryFilter(django_filters.FilterSet):
    name = django_filters.CharFilter(lookup_expr='icontains')

    class Meta:
        model  = Category
        fields = ['is_active']


class ProductFilter(django_filters.FilterSet):
    name          = django_filters.CharFilter(lookup_expr='icontains')
    price_min     = django_filters.NumberFilter(field_name='price', lookup_expr='gte')
    price_max     = django_filters.NumberFilter(field_name='price', lookup_expr='lte')
    stock_min     = django_filters.NumberFilter(field_name='stock', lookup_expr='gte')
    stock_max     = django_filters.NumberFilter(field_name='stock', lookup_expr='lte')
    category_name = django_filters.CharFilter(
        field_name='category__name', lookup_expr='icontains'
    )

    class Meta:
        model  = Product
        fields = ['is_active', 'category']
```

---

## 4.3 Serializers

```python
# store/serializers/product.py
from rest_framework import serializers
from store.models import Product
from store.serializers.category import CategorySerializer


class ProductSummarySerializer(serializers.ModelSerializer):

    class Meta:
        model  = Product
        fields = ['id', 'name', 'price', 'stock', 'is_active']


class ProductSerializer(serializers.ModelSerializer):
    category       = CategorySerializer(read_only=True)
    category_id    = serializers.PrimaryKeyRelatedField(
        source='category',
        write_only=True,
        queryset=Product.objects.none(),
    )
    price_with_tax = serializers.SerializerMethodField()
    in_stock       = serializers.SerializerMethodField()

    class Meta:
        model  = Product
        fields = [
            'id', 'name', 'description',
            'price', 'price_with_tax',
            'stock', 'in_stock', 'is_active',
            'category', 'category_id',
            'created_at', 'updated_at',
        ]
        read_only_fields = ['id', 'created_at', 'updated_at']

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        from store.models import Category
        self.fields['category_id'].queryset = Category.objects.filter(is_active=True)

    def get_price_with_tax(self, obj):
        return obj.price_with_tax

    def get_in_stock(self, obj):
        return obj.in_stock

    def validate_price(self, value):
        if value <= 0:
            raise serializers.ValidationError('Price must be greater than 0.')
        return value

    def validate_stock(self, value):
        if value < 0:
            raise serializers.ValidationError('Stock cannot be negative.')
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
from .product  import ProductSerializer, ProductSummarySerializer
```

---

## 4.4 View

```python
# store/views/product.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.permissions import IsAdminUser, AllowAny
from rest_framework.response import Response
from rest_framework.filters import SearchFilter, OrderingFilter
from django_filters.rest_framework import DjangoFilterBackend
from django.db.models import Avg, Max, Min, Sum, Count

from store.models              import Product
from store.serializers.product import ProductSerializer, ProductSummarySerializer
from store.permissions         import IsStaffOrReadOnly
from store.filters             import ProductFilter
from store.pagination          import StandardPagination


class ProductViewSet(viewsets.ModelViewSet):
    queryset           = Product.objects.select_related('category').all()
    serializer_class   = ProductSerializer
    permission_classes = [IsStaffOrReadOnly]
    pagination_class   = StandardPagination
    filter_backends    = [DjangoFilterBackend, SearchFilter, OrderingFilter]
    filterset_class    = ProductFilter
    search_fields      = ['name', 'description', 'category__name']
    ordering_fields    = ['name', 'price', 'stock', 'created_at']
    ordering           = ['name']

    @action(
        detail=True,
        methods=['post'],
        permission_classes=[IsAdminUser],
        url_path='restock',
    )
    def restock(self, request, pk=None):
        product = self.get_object()
        try:
            quantity = int(request.data.get('quantity', 0))
            if quantity <= 0:
                raise ValueError
        except (ValueError, TypeError):
            return Response(
                {'error': 'Quantity must be a positive integer.'},
                status=status.HTTP_400_BAD_REQUEST,
            )
        product.stock += quantity
        product.save(update_fields=['stock'])
        return Response({
            'id':        product.id,
            'name':      product.name,
            'new_stock': product.stock,
        })

    @action(
        detail=False,
        methods=['get'],
        permission_classes=[AllowAny],
        url_path='available',
    )
    def available(self, request):
        qs   = self.filter_queryset(
            self.get_queryset().filter(stock__gt=0, is_active=True)
        )
        page = self.paginate_queryset(qs)
        if page is not None:
            return self.get_paginated_response(
                ProductSummarySerializer(page, many=True).data
            )
        return Response(ProductSummarySerializer(qs, many=True).data)

    @action(
        detail=False,
        methods=['get'],
        url_path='stats',
    )
    def stats(self, request):
        qs      = Product.objects.all()
        active  = qs.filter(is_active=True)
        data    = active.aggregate(
            total_active  = Count('id'),
            avg_price     = Avg('price'),
            max_price     = Max('price'),
            min_price     = Min('price'),
            total_stock   = Sum('stock'),
        )
        data['total_inactive'] = qs.filter(is_active=False).count()
        data['out_of_stock']   = active.filter(stock=0).count()
        if data['avg_price']:
            data['avg_price'] = round(float(data['avg_price']), 2)
        return Response(data)
```

---

## 4.5 URLs — actualizar `store/urls.py`

```python
# store/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from rest_framework_simplejwt.views import TokenRefreshView, TokenVerifyView

from store.views.health   import health_check
from store.views.auth     import RegisterView, LogoutView
from store.views.user     import UserViewSet
from store.views.category import CategoryViewSet
from store.views.product  import ProductViewSet
from store.serializers.auth import CustomTokenView

router = DefaultRouter()
router.register('users',      UserViewSet,     basename='user')
router.register('categories', CategoryViewSet, basename='category')
router.register('products',   ProductViewSet,  basename='product')

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

## 4.6 Admin — actualizar `store/admin.py`

```python
# store/admin.py
from django.contrib import admin
from store.models import Category, Product


@admin.register(Category)
class CategoryAdmin(admin.ModelAdmin):
    list_display        = ['id', 'name', 'slug', 'is_active', 'created_at']
    list_filter         = ['is_active']
    search_fields       = ['name']
    prepopulated_fields = {'slug': ('name',)}


@admin.register(Product)
class ProductAdmin(admin.ModelAdmin):
    list_display  = ['id', 'name', 'price', 'stock', 'is_active', 'category']
    list_filter   = ['is_active', 'category']
    search_fields = ['name', 'description']
    list_editable = ['price', 'stock', 'is_active']
```

---

## 4.6.1 Actualizar `CategorySerializer` — agregar `total_products`

> **Este es el momento.** El modelo `Product` ya existe con `related_name='products'`
> en el FK a `Category`. Fue diferido desde la Etapa 3 para evitar `AttributeError`.

```python
# store/serializers/category.py — reemplazar CategorySerializer
from rest_framework import serializers
from django.utils.text import slugify
from store.models import Category


class CategorySerializer(serializers.ModelSerializer):
    total_products = serializers.SerializerMethodField()

    class Meta:
        model  = Category
        fields = [
            'id', 'name', 'slug', 'description',
            'is_active', 'total_products', 'created_at',
        ]
        read_only_fields = ['id', 'created_at']

    def get_total_products(self, obj):
        return obj.products.filter(is_active=True).count()

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

## 4.6.2 Actualizar acción `active_products` en `CategoryViewSet`

```python
# store/views/category.py — reemplazar la acción active_products
    @action(detail=True, methods=['get'], url_path='products')
    def active_products(self, request, pk=None):
        from store.models import Product
        from store.serializers.product import ProductSummarySerializer
        category = self.get_object()
        qs   = category.products.filter(is_active=True).order_by('name')
        page = self.paginate_queryset(qs)
        if page is not None:
            return self.get_paginated_response(
                ProductSummarySerializer(page, many=True).data
            )
        return Response(ProductSummarySerializer(qs, many=True).data)
```

---

## ✅ Checkpoint Etapa 4

```bash
uv run python manage.py runserver
```

| # | Endpoint | Método | Auth | Esperado |
|---|----------|--------|------|----------|
| 1 | `/api/products/` | GET | Bearer | 200 lista paginada |
| 2 | `/api/products/` | POST | Staff | 201 creado |
| 3 | `/api/products/` | POST | User | 403 forbidden |
| 4 | `/api/products/?price_min=100&price_max=500` | GET | Bearer | 200 filtrado |
| 5 | `/api/products/?stock_min=1` | GET | Bearer | 200 filtrado |
| 6 | `/api/products/?search=laptop` | GET | Bearer | 200 búsqueda |
| 7 | `/api/products/?ordering=-price` | GET | Bearer | 200 ordenado |
| 8 | `/api/products/{id}/restock/` | POST | Staff | 200 stock actualizado |
| 9 | `/api/products/available/` | GET | Público | 200 productos con stock |
| 10 | `/api/products/stats/` | GET | Bearer | 200 resumen |

---

## 📦 Colección Postman — Etapa 4

```json
{
  "info": {
    "name": "ShopAPI — Stage 4: Products",
    "_postman_id": "shopapi-stage-04",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "List products",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": "{{base_url}}/products/"
      }
    },
    {
      "name": "Filter by price range",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": {
          "raw": "{{base_url}}/products/?price_min=100&price_max=500&ordering=-price",
          "query": [
            { "key": "price_min", "value": "100" },
            { "key": "price_max", "value": "500" },
            { "key": "ordering",  "value": "-price" }
          ]
        }
      }
    },
    {
      "name": "Filter by category and stock",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": {
          "raw": "{{base_url}}/products/?category={{category_id}}&stock_min=1",
          "query": [
            { "key": "category",  "value": "{{category_id}}" },
            { "key": "stock_min", "value": "1" }
          ]
        }
      }
    },
    {
      "name": "Search product",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": {
          "raw": "{{base_url}}/products/?search=laptop",
          "query": [{ "key": "search", "value": "laptop" }]
        }
      }
    },
    {
      "name": "[Staff] Create product",
      "event": [{ "listen": "test", "script": { "exec": [
        "pm.test('Status 201', () => pm.response.to.have.status(201));",
        "pm.collectionVariables.set('product_id', pm.response.json().id);"
      ]}}],
      "request": {
        "method": "POST",
        "header": [
          { "key": "Content-Type",  "value": "application/json" },
          { "key": "Authorization", "value": "Bearer {{access}}" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"name\": \"HP Laptop 15\",\n  \"description\": \"General purpose laptop\",\n  \"price\": \"850.00\",\n  \"stock\": 10,\n  \"is_active\": true,\n  \"category_id\": {{category_id}}\n}"
        },
        "url": "{{base_url}}/products/"
      }
    },
    {
      "name": "[Staff] Update product",
      "request": {
        "method": "PATCH",
        "header": [
          { "key": "Content-Type",  "value": "application/json" },
          { "key": "Authorization", "value": "Bearer {{access}}" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"price\": \"799.00\"\n}"
        },
        "url": "{{base_url}}/products/{{product_id}}/"
      }
    },
    {
      "name": "[Staff] Restock",
      "request": {
        "method": "POST",
        "header": [
          { "key": "Content-Type",  "value": "application/json" },
          { "key": "Authorization", "value": "Bearer {{access}}" }
        ],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"quantity\": 25\n}"
        },
        "url": "{{base_url}}/products/{{product_id}}/restock/"
      }
    },
    {
      "name": "Available (public)",
      "request": {
        "method": "GET",
        "header": [],
        "url": "{{base_url}}/products/available/"
      }
    },
    {
      "name": "Stats",
      "request": {
        "method": "GET",
        "header": [{ "key": "Authorization", "value": "Bearer {{access}}" }],
        "url": "{{base_url}}/products/stats/"
      }
    }
  ],
  "variable": [
    { "key": "base_url",     "value": "http://localhost:8000/api" },
    { "key": "access",       "value": "" },
    { "key": "refresh",      "value": "" },
    { "key": "category_id",  "value": "" },
    { "key": "product_id",   "value": "" }
  ]
}
```

---

## Resumen

| Elemento | Estado |
|---|---|
| Modelo `Product` con IVA 15% + migración | ✅ |
| `ProductFilter` con rangos de precio y stock | ✅ |
| `ProductSerializer` con campos calculados | ✅ |
| `ProductSummarySerializer` para relaciones | ✅ |
| CRUD con paginación, filtros, búsqueda y ordenamiento | ✅ |
| Acciones: `restock`, `available`, `stats` | ✅ |
| Admin actualizado | ✅ |
| Colección Postman con test scripts | ✅ |

**Siguiente etapa →** CRUD de Pedidos